# Data Engineering- Complete Answers Guide

## Table of Contents
1. [Core Topics Overview](#core-topics-overview)
2. [Sample Question Answers](#sample-question-answers)
3. [Key Concepts Deep Dive](#key-concepts-deep-dive)
4. [Java Implementation Examples](#java-implementation-examples)
5. [Best Practices](#best-practices)

---

## Core Topics Overview

### Topic 1: Data Architecture & Design

**What You Need to Know:**

**ACID vs BASE:**
- **ACID** (Atomic, Consistent, Isolated, Durable): Traditional relational databases, guarantees strong consistency. Best for financial transactions where accuracy is critical.
- **BASE** (Basically Available, Soft state, Eventually consistent): Used in distributed systems (NoSQL). Data might not be immediately consistent but will be eventually. Best for high-availability systems like social media feeds.

**Schema Evolution & Backward Compatibility:**
- Add nullable columns instead of required ones
- Use versioning in API responses (v1.0, v2.0)
- Support multiple versions of schema simultaneously
- Use Apache Avro or Protocol Buffers for schema management

**Data Versioning:**
- Maintain metadata about data versions (creation date, modification date, version number)
- Use version numbers in data names (table_v1, table_v2)
- Keep audit trails of what changed and when

**Star Schema vs Normalized vs Denormalized:**
- **Star Schema**: Facts at center, dimensions around. Used in data warehouses. Easy to query but requires more storage.
- **Normalized**: Multiple tables, minimal redundancy. Used in transactional systems. Saves space but requires joins.
- **Denormalized**: Data combined in fewer tables, redundancy accepted. Used in analytics. Faster queries but harder to maintain.

---

### Topic 2: Database & Query Performance

**SQL Query Optimization:**
1. Use EXPLAIN PLAN to understand query execution
2. Add appropriate indexes (B-tree for range queries, Hash for equality)
3. Avoid SELECT * - fetch only needed columns
4. Use WHERE clause to filter early
5. Use INNER JOIN instead of WHERE clause filtering when possible
6. Partition large tables for faster scans

**Index Types:**
- **B-tree**: Good for range queries (>, <, BETWEEN), ORDER BY. Default for most databases.
- **Hash**: Excellent for equality searches (=), not for ranges
- **Bitmap**: For low-cardinality columns (gender: M/F)
- **Composite**: Multiple columns, useful for queries using all indexed columns

**N+1 Query Problem:**
```
Problem: 
for (User user : users) {
    Address address = fetchAddressById(user.getId()); // N queries for N users
}

Solution 1 - Batch Fetch:
List<Address> addresses = fetchAddressesByIds(userIds); // 1 query

Solution 2 - JOIN:
SELECT u.*, a.* FROM users u JOIN addresses a ON u.id = a.user_id;

Solution 3 - Eager Loading (Hibernate):
@ManyToOne(fetch = FetchType.EAGER)
private Address address;
```

**Explain Plans:**
```
Sample Output:
EXPLAIN SELECT * FROM orders WHERE status = 'PENDING';
Output:
Table Scan on orders (rows: 10000)
  Filter: status = 'PENDING' (estimated rows: 100)
Cost: 150 (Full table scan)

With Index:
Index Scan on idx_orders_status (rows: 100)
Cost: 5 (Direct index access)
```

---

### Topic 3: Data Pipelines & Integration

**ETL vs ELT:**
- **ETL** (Extract, Transform, Load): Transform data in middleware before loading. Traditional approach. Good when target system has limited resources.
- **ELT** (Extract, Load, Transform): Load raw data, transform in destination. Modern cloud approach. Leverages power of cloud data warehouses.

**Data Quality Frameworks:**
```
Quality Checks:
1. Completeness: Are all required fields populated?
2. Uniqueness: Are primary/unique keys truly unique?
3. Consistency: Does data match across systems?
4. Validity: Is data in correct format and range?
5. Accuracy: Does data represent real-world values?
6. Timeliness: Is data current and available when needed?

Implementation:
- Data profiling before ETL
- Validation rules at each stage
- DQ metrics dashboard
- Automated alerts for failures
```

**Idempotency in Data Processing:**
```
Non-Idempotent:
update users set balance = balance + 100; // Running twice doubles the transfer

Idempotent:
update users set balance = balance + 100 
where user_id = 123 and transaction_id not in (select transaction_id from processed);

Or:
insert into processed (transaction_id) values (123);
-- Safe to re-run, won't duplicate
```

**Change Data Capture (CDC):**
- Capture INSERT, UPDATE, DELETE operations on source system
- Methods: Query-based (timestamp columns), Log-based (transaction logs), Query+Comparison
- Tools: Debezium, Oracle GoldenGate, AWS DMS
- Output typically to Kafka topics for real-time pipeline consumption

---

### Topic 4: Big Data Technologies

**Spark Architecture:**
```
Driver Program
    |
Cluster Manager (Spark, YARN, Kubernetes)
    |
Executor 1, Executor 2, Executor N

Data Flow:
- RDD: Immutable distributed collection (low-level)
- DataFrame: Structured data in rows/columns (high-level, optimized)
- Dataset: Type-safe, compiled collection (Scala/Java)
```

**MapReduce Fundamentals:**
```
Map Phase: (key, value) -> (intermediate_key, intermediate_value)
Shuffle & Sort: Group by intermediate_key
Reduce Phase: (intermediate_key, [values]) -> final_result

Example: Word Count
Map: "hello world" -> (hello, 1), (world, 1)
Shuffle: (hello, [1]), (world, [1])
Reduce: (hello, [1]) -> (hello, 1)
```

**Kafka Architecture:**
```
Producers -> Broker (Topics/Partitions) -> Consumers

Guarantees:
- At-most-once: Message might be lost
- At-least-once: Message might be duplicated
- Exactly-once: Requires idempotent processing
```

**Streaming vs Batch Trade-offs:**
- **Batch**: Higher latency, better for bulk processing, easier to debug, better resource efficiency
- **Streaming**: Lower latency (seconds/milliseconds), complex state management, harder debugging
- **Micro-batching**: Hybrid approach, Spark Streaming runs batches every X seconds

---

### Topic 5: Data Warehousing & Lakes

**Fact vs Dimension Tables:**
```
Fact Table:
- Stores measured events (sales, clicks, views)
- Contains foreign keys to dimensions
- Contains numeric metrics (amount, quantity)
- Many rows, narrow
- Example: SALES_FACT (order_date, product_id, customer_id, amount)

Dimension Table:
- Stores descriptive attributes
- Denormalized for query performance
- Slower changing
- Example: PRODUCT_DIM (product_id, name, category, price)
```

**Slowly Changing Dimensions (SCD):**
```
Type 1: Overwrite
- Old data lost
- Simple to implement
- Use: Stock prices, interest rates

Type 2: Add new row with effective dates
- Keep history
- More complex
- Use: Customer addresses, salary history
Example:
CustomerID | Dimension | Value      | Effective_From | Effective_To
1          | Address   | 123 Main   | 2023-01-01     | 2024-01-01
1          | Address   | 456 Oak    | 2024-01-01     | 9999-12-31

Type 3: Add new column
- Keep previous value in separate column
- Limited history
- Use: Product status changes
Example:
CustomerID | Address_Current | Address_Previous
1          | 456 Oak         | 123 Main
```

**Data Modeling for Analytics (Kimball vs Inmon):**
- **Kimball**: Dimensional modeling, star schema, business-focused, easy to understand
- **Inmon**: 3NF normalization, data vault, technical, more normalized

**Medallion Architecture (Bronze, Silver, Gold):**
```
Bronze Layer: Raw data as-is from sources
- Minimal transformation
- No cleaning
- Example: Raw JSON files from APIs

Silver Layer: Cleaned, deduplicated, validated data
- Remove duplicates
- Type casting
- Handle nulls
- Example: Normalized tables with business rules applied

Gold Layer: Business-ready datasets
- Aggregations for specific use cases
- Optimized for queries
- Example: Customer 360 view, dashboard tables
```

---

### Topic 6: AWS Data Services

**RDS vs DynamoDB:**
```
RDS (Relational):
- Structured data with relationships
- Complex queries (JOINs, aggregations)
- ACID transactions required
- Predictable access patterns
- Example: ERP systems, financial records

DynamoDB (NoSQL):
- Semi-structured or unstructured data
- Key-value access patterns
- High throughput needed
- Flexible schema
- Example: User profiles, real-time gaming leaderboards
```

**Multi-AZ and Read Replicas:**
```
Multi-AZ (High Availability):
- Primary in AZ1, Standby in AZ2
- Automatic failover
- Synchronous replication
- For disaster recovery

Read Replicas (Scalability):
- Asynchronous replication
- Can be in same or different region
- For read-heavy workloads
- Manual failover required
```

**S3 Partitioning Schemes:**
```
Bad Partitioning:
s3://bucket/data.csv (all data in one file)

Good Partitioning:
s3://bucket/year=2024/month=03/day=06/data.parquet
Benefits:
- Partition pruning: Only read relevant partitions
- Parallel processing: Each partition processed independently
- Cost: Pay only for data queried
```

**Redshift for Analytics:**
- Column-oriented database (compress well, scan only needed columns)
- Massive parallel processing (MPP)
- Distribution keys and sort keys important
- Compression algorithms (AZ64, ZSTD)

**Security in Data Applications:**
```
Encryption:
- At rest: S3 server-side encryption (SSE-S3, SSE-KMS)
- In transit: TLS/SSL for data movement
- In use: Application-level encryption for sensitive fields

Access Control:
- IAM roles for AWS service authentication
- VPC for network isolation
- Security groups for port-level control
- RLS (Row-Level Security) for data filtering

Compliance:
- Encryption of sensitive data (credit cards, SSN)
- Audit logging (CloudTrail)
- Data retention policies
```

---

### Topic 7: Performance Tuning & Monitoring

**Key Metrics:**
```
Database Metrics:
- Query response time (p50, p95, p99)
- Throughput (queries/second)
- Cache hit ratio (ideally >95%)
- Connection count vs pool size
- Lock wait time
- Full table scan frequency

Application Metrics:
- Request latency
- Error rate
- CPU/Memory usage
- Garbage collection time (Java)
- Thread pool utilization
```

**Connection Pooling:**
```
Why: Creating DB connections is expensive
What: Maintain pool of reusable connections

Sizing:
- Too small: Thread starvation, slow response
- Too large: Memory waste, database overload
- Rule of thumb: min = (cores * 2) + overhead, max = (cores * 4)
- Example: 8-core system, min=18, max=32

Popular libraries:
- HikariCP: Lightweight, fast (recommended)
- c3p0: Full-featured
- DBCP2: Apache implementation

Configuration:
hikari:
  connection-timeout: 30000
  idle-timeout: 600000
  max-lifetime: 1800000
  maximum-pool-size: 20
```

**Database Deadlocks:**
```
Cause: Circular wait for resources
Example:
Transaction 1: Lock A, wait for B
Transaction 2: Lock B, wait for A

Solutions:
1. Lock ordering: Always acquire in same order
2. Timeout: Transaction automatically rolls back after timeout
3. Deadlock detection: DB detects and aborts one transaction
4. Minimize transaction scope: Lock for shorter duration
5. Isolation level: Lower isolation = fewer locks (but consistency trade-off)
```

**Caching Strategies:**
```
Cache-Aside (Lazy Loading):
app -> cache miss -> fetch from DB -> cache -> return
Benefits: Only cache accessed data
Drawback: Initial hit always slow

Write-Through:
write -> cache -> DB -> return
Benefits: Cache always up-to-date
Drawback: Write latency

Write-Behind:
write -> cache -> return, async write to DB
Benefits: Fast writes
Drawback: Data loss if cache failure before sync

TTL (Time-To-Live):
- Set expiration time on cache entries
- Balance between freshness and hit rate
- Example: Cache user session 30 minutes
```

---

### Topic 8: Metadata Management & Governance

**Data Lineage:**
```
Tracking: data source -> transformations -> final dataset
Purpose: Understand data flow, impact analysis, debugging

Example:
Raw Customer CSV
    |
Cleanse (remove nulls)
    |
Enrich (add demographics)
    |
Gold Layer Customer Dataset
    |
BI Dashboard

Questions answered:
- Where did this value come from?
- What datasets depend on this table?
- What transformations were applied?
```

**Data Catalogs:**
- Metadata repositories (Collibra, Alation)
- Search and discover datasets
- Data ownership, quality metrics
- Uses: Reduce redundant datasets, find relevant data, governance

**Data Quality Metrics:**
```
Business Metrics:
- % records with null values
- Distribution of values (outliers)
- Freshness (age of latest record)
- Volume consistency (expected vs actual rows)

Technical Metrics:
- Invalid formats
- Constraint violations (duplicates, foreign key)
- Referential integrity
- Schema conformance
```

**RBAC (Role-Based Access Control):**
```
Example:
SELECT * FROM users; -- Only Analytics Read role can see

Roles:
- Data Owner: Full access, governance
- Data Engineer: Write access, transformation
- Analytics: Read-only
- Admin: All permissions

Database Implementation:
CREATE ROLE analyst;
GRANT SELECT ON schema.* TO analyst;
CREATE USER john IDENTIFIED BY password;
GRANT analyst TO john;
```

---

## Sample Question Answers

### Easy-Medium Level Questions

#### Q1: Design a Java application that reads from a data pipeline daily. How would you handle failures and retries?

**Answer:**

**Architecture:**
```
Scheduler (Quartz/Spring Scheduler)
    |
Read from Data Source
    |
Process/Transform
    |
Write to Target
    |
Error Handling
    |
Logging & Monitoring
```

**Implementation:**
```java
@Configuration
@EnableScheduling
public class DataPipelineConfig {
    
    @Scheduled(cron = "0 2 * * *") // 2 AM daily
    public void runDailyDataPipeline() {
        int maxRetries = 3;
        int retryCount = 0;
        
        while (retryCount < maxRetries) {
            try {
                // Read from source
                List<DataRecord> records = readFromDataSource();
                
                // Transform
                List<DataRecord> processed = transform(records);
                
                // Load to target
                writeToTarget(processed);
                
                // Success
                logger.info("Pipeline completed successfully");
                return;
                
            } catch (TransientException e) {
                retryCount++;
                if (retryCount < maxRetries) {
                    long backoffTime = (long) (1000 * Math.pow(2, retryCount)); // Exponential backoff
                    Thread.sleep(backoffTime);
                } else {
                    logger.error("Pipeline failed after {} retries", maxRetries);
                    sendAlert("Pipeline failure alert");
                    throw new DataPipelineException("Failed to complete pipeline", e);
                }
            }
        }
    }
    
    private List<DataRecord> readFromDataSource() {
        // Read from CSV, API, database, etc.
        // Implement pagination for large datasets
        try (BufferedReader reader = new BufferedReader(new FileReader("data.csv"))) {
            return reader.lines()
                .map(this::parseRecord)
                .collect(Collectors.toList());
        }
    }
    
    private List<DataRecord> transform(List<DataRecord> records) {
        return records.stream()
            .filter(r -> isValid(r)) // Validation
            .map(r -> enrichData(r)) // Enrichment
            .collect(Collectors.toList());
    }
    
    private void writeToTarget(List<DataRecord> records) {
        // Write in batches to database
        int batchSize = 1000;
        for (int i = 0; i < records.size(); i += batchSize) {
            List<DataRecord> batch = records.subList(i, 
                Math.min(i + batchSize, records.size()));
            repository.saveAll(batch);
        }
    }
}
```

**Failure Handling:**
1. **Exponential Backoff**: Wait longer between retries
2. **Circuit Breaker Pattern**: Stop retrying after threshold
3. **Dead Letter Queue**: Send failed records to separate queue for manual review
4. **Idempotent Operations**: Safe to retry without duplicates

---

#### Q2: How would you optimize a slow running query from Java without modifying the database?

**Answer:**

**Techniques:**

1. **Fetch Only Needed Columns:**
```java
// Bad
query = "SELECT * FROM orders";

// Good
query = "SELECT order_id, customer_id, order_date, total FROM orders";
```

2. **Use Pagination:**
```java
public Page<Order> getOrdersWithPagination(int pageNumber, int pageSize) {
    Pageable pageable = PageRequest.of(pageNumber, pageSize);
    return orderRepository.findAll(pageable);
}

// Fetches only 20 records instead of 1 million
```

3. **Add Query Filters:**
```java
// Bad: Fetch all then filter
List<Order> all = repository.findAll();
List<Order> recent = all.stream()
    .filter(o -> o.getOrderDate().isAfter(date))
    .collect(Collectors.toList());

// Good: Filter at database level
List<Order> recent = repository.findOrdersAfter(date);
```

4. **Use Batch Operations:**
```java
// Bad: N separate queries
for (Order order : orders) {
    repository.save(order);
}

// Good: Batch insert
repository.saveAll(orders); // Single transaction
```

5. **Implement Caching:**
```java
@Service
public class OrderService {
    
    @Cacheable(value = "orders", key = "#orderId")
    public Order getOrder(Long orderId) {
        return repository.findById(orderId);
    }
    
    @CacheEvict(value = "orders", key = "#order.id")
    public void updateOrder(Order order) {
        repository.save(order);
    }
}
```

6. **Use Lazy Loading Carefully:**
```java
// Bad: Eager fetch all related entities
@Entity
public class Order {
    @ManyToOne(fetch = FetchType.EAGER)
    private Customer customer;
}

// Good: Load on demand
@Entity
public class Order {
    @ManyToOne(fetch = FetchType.LAZY)
    private Customer customer;
}

// Then explicitly fetch when needed
Order order = repository.findWithCustomer(orderId);
```

7. **Connection Pooling:**
```yaml
# Application.yml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 30000
      idle-timeout: 600000
```

---

#### Q3: Explain how you'd implement pagination for a large dataset in a REST API.

**Answer:**

**Offset-Based Pagination (Traditional):**
```java
@RestController
@RequestMapping("/api/orders")
public class OrderController {
    
    @GetMapping
    public ResponseEntity<Page<OrderDTO>> getOrders(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size,
            @RequestParam(defaultValue = "id,desc") String sort) {
        
        Sort.Direction direction = sort.endsWith(",asc") ? Sort.Direction.ASC : Sort.Direction.DESC;
        String sortBy = sort.split(",")[0];
        
        Pageable pageable = PageRequest.of(page, size, Sort.by(direction, sortBy));
        Page<Order> orders = orderRepository.findAll(pageable);
        
        return ResponseEntity.ok(orders.map(this::convertToDTO));
    }
}

// Usage: GET /api/orders?page=0&size=20&sort=createdAt,desc
```

**Cursor-Based Pagination (Better for large datasets):**
```java
@GetMapping
public ResponseEntity<PagedResponse<OrderDTO>> getOrdersCursorBased(
        @RequestParam(required = false) Long cursor,
        @RequestParam(defaultValue = "20") int size) {
    
    List<Order> orders = orderRepository.findNextOrders(cursor, size + 1);
    
    boolean hasMore = orders.size() > size;
    if (hasMore) {
        orders = orders.subList(0, size);
    }
    
    Long nextCursor = hasMore ? orders.get(orders.size() - 1).getId() : null;
    
    return ResponseEntity.ok(new PagedResponse<>(
        orders.stream().map(this::convertToDTO).collect(Collectors.toList()),
        nextCursor,
        hasMore
    ));
}

// Query
@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {
    @Query("SELECT o FROM Order o WHERE (:cursor IS NULL OR o.id > :cursor) " +
           "ORDER BY o.id ASC LIMIT :limit")
    List<Order> findNextOrders(@Param("cursor") Long cursor, @Param("limit") int limit);
}

// Usage: GET /api/orders/cursor?cursor=100&size=20
```

**Response Structure:**
```json
{
  "data": [
    {"id": 101, "customer": "John", "amount": 500},
    {"id": 102, "customer": "Jane", "amount": 600}
  ],
  "pagination": {
    "nextCursor": 102,
    "hasMore": true,
    "size": 2
  }
}
```

**Comparison:**
- **Offset**: Simple, but slow with large offsets (needs to skip n records)
- **Cursor**: Faster, better for real-time data, but requires ordering column

---

#### Q4: What's the difference between connection pooling with size 10 vs 100? When would you use each?

**Answer:**

**Connection Pooling Basics:**
- Maintaining connections to database is expensive
- Pool keeps connections ready for reuse
- Active connections limited by pool size

**Size 10 vs 100:**

```
Pool Size 10:
- Max 10 concurrent database queries
- Memory efficient (each connection ~1-2 MB)
- Suitable for: Low to medium traffic applications
- Risk: Thread starvation if 11+ requests arrive simultaneously

Pool Size 100:
- Max 100 concurrent queries
- Memory usage ~100-200 MB just for connections
- Suitable for: High-traffic applications
- Risk: Database server overload, slow performance
```

**Sizing Formula:**
```
For 8-core system:
- Minimum: (8 * 2) + 2 = 18
- Maximum: (8 * 4) - 1 = 31
- Typical: 20-25
```

**When to Use Each:**

```
Use 10:
- Monolithic application
- Low concurrent users (<100)
- Limited server resources
- Development/testing environment
- Example: Internal tool with <10 simultaneous users

Use 100:
- Microservices architecture
- High concurrent users (>1000)
- Dedicated database server
- Each service instance gets 100 connections
- Example: E-commerce website
```

**Database Perspective:**
```
Single Database:
- If 20 app instances × 100 pool size = 2000 connections
- Most databases support 1000-5000 connections
- Need to limit or use connection pooling on database side too

Example Configuration:
hikari:
  minimum-idle: 10        # Keep ready
  maximum-pool-size: 20   # Max concurrent
  connection-timeout: 30s # Wait time
  idle-timeout: 10m       # Close idle connections
```

---

#### Q5: How do you handle NULL values in your application logic?

**Answer:**

**Database Level:**
```sql
-- Bad: Accept NULLs everywhere
CREATE TABLE users (
    id INT,
    name VARCHAR(100),
    email VARCHAR(100),
    phone VARCHAR(20)
);

-- Good: Define constraints
CREATE TABLE users (
    id INT NOT NULL,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(100) NOT NULL,
    phone VARCHAR(20), -- Optional
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP
);
```

**Java Application Level:**

1. **Null Checks:**
```java
// Old way
if (user != null && user.getAddress() != null) {
    String city = user.getAddress().getCity();
}

// Modern way with Optional
Optional<String> city = Optional.ofNullable(user)
    .map(User::getAddress)
    .map(Address::getCity);
    
String cityName = city.orElse("Unknown");
```

2. **Null-Safe Method Calls:**
```java
// Using Optional chaining
List<Order> orders = Optional.ofNullable(customer)
    .map(Customer::getOrders)
    .orElse(Collections.emptyList());

// Using Stream
orders.stream()
    .filter(Objects::nonNull)
    .map(Order::getAmount)
    .reduce(BigDecimal.ZERO, BigDecimal::add);
```

3. **Validation & Annotations:**
```java
@Entity
public class Order {
    
    @NotNull(message = "Order ID cannot be null")
    @Id
    private Long orderId;
    
    @NotNull(message = "Customer is required")
    @ManyToOne
    private Customer customer;
    
    @NotBlank(message = "Status cannot be empty")
    private String status;
    
    @DecimalMin(value = "0.01", message = "Amount must be positive")
    private BigDecimal amount;
}

// In Controller
@PostMapping
public ResponseEntity<Order> createOrder(@Valid @RequestBody Order order) {
    // order is guaranteed to be non-null and valid
    return ResponseEntity.ok(orderService.save(order));
}
```

4. **Handling in Queries:**
```java
// Counting non-null values
@Query("SELECT COUNT(o) FROM Order o WHERE o.customerId IS NOT NULL")
long countNonNullCustomers();

// Coalesce in queries
@Query("SELECT COALESCE(o.discount, 0) FROM Order o WHERE o.id = :id")
BigDecimal getDiscount(@Param("id") Long id);

// In Java
BigDecimal discount = order.getDiscount() != null ? order.getDiscount() : BigDecimal.ZERO;
```

5. **Default Values:**
```java
@Entity
public class Order {
    
    @NotNull
    @Column(columnDefinition = "VARCHAR(20) DEFAULT 'PENDING'")
    private String status = "PENDING";
    
    @Column(columnDefinition = "TIMESTAMP DEFAULT CURRENT_TIMESTAMP")
    private LocalDateTime createdAt = LocalDateTime.now();
    
    @Column(columnDefinition = "DECIMAL(10,2) DEFAULT 0.00")
    private BigDecimal discount = BigDecimal.ZERO;
}
```

6. **Null-Safe toString() and equals():**
```java
String result = Objects.toString(user, "No user data");
boolean equals = Objects.equals(obj1, obj2); // Null-safe
int hash = Objects.hash(user, order); // Null-safe hashing
```

**Best Practices:**
- ✅ Define NOT NULL constraints in database
- ✅ Use Optional for nullable references
- ✅ Validate input with annotations
- ✅ Provide meaningful defaults
- ✅ Document which fields can be null
- ❌ Avoid unchecked null dereferences
- ❌ Don't use null as sentinel value

---

### Medium-Hard Level Questions

#### Q6: Design a system where Java applications feed data into a data lake on S3. How do you ensure data quality?

**Answer:**

**System Architecture:**
```
Java Application
    |
Data Validation Layer
    |
Data Transformation
    |
Quality Checks
    |
S3 Lake (Bronze -> Silver -> Gold)
    |
Metadata Store (DynamoDB)
```

**Implementation:**

```java
@Service
public class DataLakeService {
    
    @Autowired
    private AmazonS3 s3Client;
    
    @Autowired
    private DataQualityValidator validator;
    
    @Autowired
    private MetadataRepository metadataRepo;
    
    // Configuration
    private static final String BUCKET_NAME = "data-lake";
    private static final String BRONZE_PATH = "bronze/";
    private static final String SILVER_PATH = "silver/";
    
    public void ingestData(List<DataRecord> records, String source, LocalDate date) {
        try {
            // Step 1: Validate input data
            ValidationResult validation = validator.validate(records);
            if (!validation.isValid()) {
                logQualityIssues(validation);
                sendAlert("Data quality issues", validation.getErrors());
            }
            
            // Step 2: Upload raw data to Bronze layer
            String bronzeKey = String.format("%s%s/%s/data_%s.parquet",
                BRONZE_PATH, source, date, System.currentTimeMillis());
            uploadToS3(records, bronzeKey);
            
            // Step 3: Transform and deduplicate for Silver layer
            List<DataRecord> cleanedRecords = transformData(records);
            
            // Step 4: Apply SCD Type 2 for slowly changing dimensions
            List<DataRecord> dimensionRecords = handleSlowlyChangingDimensions(cleanedRecords);
            
            // Step 5: Quality checks on Silver layer
            QualityMetrics metrics = calculateQualityMetrics(dimensionRecords);
            
            String silverKey = String.format("%s%s/%s/data_%s.parquet",
                SILVER_PATH, source, date, System.currentTimeMillis());
            uploadToS3(dimensionRecords, silverKey);
            
            // Step 6: Store metadata
            recordMetadata(source, date, bronzeKey, silverKey, metrics);
            
            logger.info("Successfully ingested data: {} records from {}", 
                records.size(), source);
            
        } catch (Exception e) {
            logger.error("Data ingestion failed", e);
            throw new DataLakeException("Failed to ingest data into lake", e);
        }
    }
    
    private ValidationResult validate(List<DataRecord> records) {
        ValidationResult result = new ValidationResult();
        
        for (DataRecord record : records) {
            // Check completeness
            if (record.getMandatoryFields().stream().anyMatch(f -> f == null)) {
                result.addError("Incomplete record: " + record.getId());
            }
            
            // Check validity (format, range)
            if (!isValidFormat(record.getEmail())) {
                result.addError("Invalid email format: " + record.getEmail());
            }
            
            if (record.getAge() < 0 || record.getAge() > 150) {
                result.addError("Invalid age: " + record.getAge());
            }
            
            // Check uniqueness (against existing data)
            if (isDuplicate(record)) {
                result.addError("Duplicate record: " + record.getId());
            }
        }
        
        return result;
    }
    
    private List<DataRecord> transformData(List<DataRecord> records) {
        return records.stream()
            // Remove duplicates (keep first occurrence)
            .filter(new DuplicateFilter<>())
            // Trim whitespace from strings
            .map(r -> r.trimStrings())
            // Convert dates to standard format
            .map(r -> r.normalizeDateFormats())
            // Remove sensitive data
            .map(r -> r.maskSensitiveFields(Arrays.asList("ssn", "credit_card")))
            // Standardize enums
            .map(r -> r.standardizeEnums())
            .collect(Collectors.toList());
    }
    
    private QualityMetrics calculateQualityMetrics(List<DataRecord> records) {
        int total = records.size();
        
        long nullCount = records.stream()
            .filter(r -> r.hasNullFields())
            .count();
        
        long invalidFormatCount = records.stream()
            .filter(r -> !isValidFormat(r))
            .count();
        
        long duplicateCount = records.stream()
            .filter(this::isDuplicate)
            .count();
        
        return QualityMetrics.builder()
            .totalRecords(total)
            .nullRecords(nullCount)
            .completeness((double)(total - nullCount) / total)
            .invalidFormatRecords(invalidFormatCount)
            .duplicateRecords(duplicateCount)
            .accuracy((double)(total - invalidFormatCount - duplicateCount) / total)
            .qualityScore(calculateQualityScore(nullCount, invalidFormatCount, duplicateCount, total))
            .build();
    }
    
    private void uploadToS3(List<DataRecord> records, String key) {
        // Convert to Parquet format (columnar, compressed)
        byte[] parquetData = convertToParquet(records);
        
        ObjectMetadata metadata = new ObjectMetadata();
        metadata.setContentLength(parquetData.length);
        metadata.setContentType("application/octet-stream");
        metadata.addUserMetadata("source", "data-lake-service");
        metadata.addUserMetadata("timestamp", LocalDateTime.now().toString());
        
        PutObjectRequest request = new PutObjectRequest(BUCKET_NAME, key,
            new ByteArrayInputStream(parquetData), metadata);
        
        // Add server-side encryption
        request.setServerSideEncryption(ObjectMetadata.AES_256_SERVER_SIDE_ENCRYPTION);
        
        s3Client.putObject(request);
    }
    
    private void recordMetadata(String source, LocalDate date, String bronzeKey,
                              String silverKey, QualityMetrics metrics) {
        DataLakeMetadata metadata = DataLakeMetadata.builder()
            .id(UUID.randomUUID().toString())
            .source(source)
            .ingestionDate(date)
            .bronzeLocation(bronzeKey)
            .silverLocation(silverKey)
            .recordCount(metrics.getTotalRecords())
            .completenessScore(metrics.getCompleteness())
            .accuracyScore(metrics.getAccuracy())
            .qualityScore(metrics.getQualityScore())
            .ingestionTimestamp(LocalDateTime.now())
            .status("SUCCESS")
            .build();
        
        metadataRepo.save(metadata);
    }
}
```

**Quality Metrics Model:**
```java
@Data
@Builder
public class QualityMetrics {
    private int totalRecords;
    private long nullRecords;
    private double completeness; // 0-1
    private long invalidFormatRecords;
    private long duplicateRecords;
    private double accuracy; // 0-1
    private double qualityScore; // 0-100
    
    public boolean isAcceptableQuality() {
        return qualityScore >= 90.0; // 90% quality threshold
    }
}
```

**Data Lake Partitioning:**
```
s3://data-lake/
├── bronze/
│   ├── source_system_1/
│   │   ├── 2024/03/06/
│   │   │   └── data_1709762400000.parquet (raw data)
│   │   └── 2024/03/05/
│   └── source_system_2/
├── silver/
│   ├── source_system_1/
│   │   ├── 2024/03/06/
│   │   │   └── data_cleaned_1709762400000.parquet (transformed, deduplicated)
│   │   └── 2024/03/05/
│   └── source_system_2/
└── metadata/
    └── data_lineage.json (tracks lineage)
```

**Monitoring & Alerting:**
```yaml
# Prometheus metrics
- name: data_lake_quality_score
  description: Quality score for ingested data (0-100)
  threshold: 90
  alert_level: warning

- name: data_completeness
  description: Percentage of non-null fields
  threshold: 95%
  alert_level: warning

- name: data_ingestion_latency
  description: Time to ingest data
  threshold: 5 minutes
  alert_level: critical
```

---

#### Q7: Explain how you'd implement an efficient ETL process triggered from a Java application.

**Answer:**

**ETL Process Design:**

```
Extract Phase: Read data from sources
    ↓
Transform Phase: Clean, enrich, aggregate
    ↓
Load Phase: Write to target system
    ↓
Monitoring: Track performance and quality
```

**Implementation:**

```java
@Service
@Transactional
public class ETLProcessService {
    
    @Autowired
    private DataExtractor extractor;
    
    @Autowired
    private DataTransformer transformer;
    
    @Autowired
    private DataLoader loader;
    
    @Autowired
    private ETLMetricsService metricsService;
    
    @Value("${etl.batch-size:10000}")
    private int batchSize;
    
    @Scheduled(cron = "0 0 * * * ?") // Hourly
    public void executeETL() {
        ETLExecution execution = new ETLExecution();
        LocalDateTime startTime = LocalDateTime.now();
        
        try {
            // Phase 1: Extract
            logger.info("Starting EXTRACT phase");
            List<RawRecord> rawData = extract();
            execution.setExtractedCount(rawData.size());
            metricsService.recordMetric("etl.extract.record_count", rawData.size());
            
            // Phase 2: Transform (in batches for memory efficiency)
            logger.info("Starting TRANSFORM phase");
            int transformedCount = transformInBatches(rawData, execution);
            execution.setTransformedCount(transformedCount);
            metricsService.recordMetric("etl.transform.record_count", transformedCount);
            
            // Phase 3: Load
            logger.info("Starting LOAD phase");
            int loadedCount = loadInBatches(execution.getTransformedData());
            execution.setLoadedCount(loadedCount);
            metricsService.recordMetric("etl.load.record_count", loadedCount);
            
            // Phase 4: Verification
            int matchCount = verifyIntegrity(execution);
            execution.setVerifiedCount(matchCount);
            
            // Record execution
            execution.setStatus("SUCCESS");
            execution.setDuration(Duration.between(startTime, LocalDateTime.now()));
            execution.setCompletedTime(LocalDateTime.now());
            recordExecution(execution);
            
            logger.info("ETL completed successfully! Extracted: {}, Transformed: {}, Loaded: {}",
                rawData.size(), transformedCount, loadedCount);
            
        } catch (Exception e) {
            execution.setStatus("FAILED");
            execution.setErrorMessage(e.getMessage());
            execution.setCompletedTime(LocalDateTime.now());
            recordExecution(execution);
            
            logger.error("ETL process failed", e);
            sendAlert("ETL Failure", "ETL process failed: " + e.getMessage());
            throw new ETLException("ETL execution failed", e);
        }
    }
    
    // EXTRACT Phase
    private List<RawRecord> extract() {
        List<RawRecord> allData = new ArrayList<>();
        
        // Extract from multiple sources in parallel
        CompletableFuture<List<RawRecord>> source1 = 
            CompletableFuture.supplyAsync(() -> extractor.extractFromDatabaseSource());
        
        CompletableFuture<List<RawRecord>> source2 = 
            CompletableFuture.supplyAsync(() -> extractor.extractFromAPISource());
        
        CompletableFuture<List<RawRecord>> source3 = 
            CompletableFuture.supplyAsync(() -> extractor.extractFromS3Source());
        
        // Wait for all to complete
        CompletableFuture<Void> all = CompletableFuture.allOf(source1, source2, source3);
        all.join();
        
        try {
            allData.addAll(source1.get());
            allData.addAll(source2.get());
            allData.addAll(source3.get());
        } catch (Exception e) {
            throw new ETLException("Failed to extract data", e);
        }
        
        return allData;
    }
    
    // TRANSFORM Phase
    private int transformInBatches(List<RawRecord> rawData, ETLExecution execution) {
        int totalTransformed = 0;
        
        for (int i = 0; i < rawData.size(); i += batchSize) {
            int end = Math.min(i + batchSize, rawData.size());
            List<RawRecord> batch = rawData.subList(i, end);
            
            // Transform batch
            List<TransformedRecord> transformed = batch.stream()
                .filter(r -> validator.isValid(r))
                .map(this::enrich)
                .map(this::denormalize)
                .map(this::aggregate)
                .filter(r -> r != null) // Remove invalid transformations
                .collect(Collectors.toList());
            
            totalTransformed += transformed.size();
            execution.addTransformedData(transformed);
            
            // Track invalid records
            int invalid = batch.size() - transformed.size();
            if (invalid > 0) {
                metricsService.recordMetric("etl.transform.invalid_records", invalid);
                logger.warn("Invalid records in batch: {}/{}", invalid, batch.size());
            }
        }
        
        return totalTransformed;
    }
    
    private RawRecord enrich(RawRecord record) {
        // Add additional data from lookup tables
        String masterData = getMasterData(record.getId());
        record.setMasterData(masterData);
        return record;
    }
    
    private TransformedRecord denormalize(RawRecord record) {
        // For performance, denormalize related lookups
        TransformedRecord transformed = new TransformedRecord();
        transformed.setId(record.getId());
        transformed.setName(record.getName());
        transformed.setCategory(record.getCategoryName()); // Pre-fetched
        transformed.setRegionCode(record.getRegionCode());
        return transformed;
    }
    
    private TransformedRecord aggregate(TransformedRecord record) {
        // Apply business logic aggregations
        record.setTotalSpent(calculateTotal(record.getId()));
        record.setCustomerSegment(segmentCustomer(record));
        return record;
    }
    
    // LOAD Phase (Upsert for idempotency)
    private int loadInBatches(List<TransformedRecord> data) {
        int totalLoaded = 0;
        
        for (int i = 0; i < data.size(); i += batchSize) {
            int end = Math.min(i + batchSize, data.size());
            List<TransformedRecord> batch = data.subList(i, end);
            
            // Use batch upsert for idempotency
            totalLoaded += loader.upsertBatch(batch);
            
            metricsService.recordMetric("etl.load.batch_loaded", batch.size());
            logger.info("Loaded batch: {}/{}", end, data.size());
        }
        
        return totalLoaded;
    }
    
    // VERIFY Phase
    private int verifyIntegrity(ETLExecution execution) {
        int expected = execution.getTransformedCount();
        int actual = loader.countByLoadTime(execution.getCompletedTime());
        
        if (Math.abs(expected - actual) <= (expected * 0.01)) { // 1% tolerance
            logger.info("Verification passed: {} records verified", actual);
            return actual;
        } else {
            logger.error("Verification failed: Expected {}, found {}", expected, actual);
            throw new ETLException("Data integrity check failed");
        }
    }
    
    private void recordExecution(ETLExecution execution) {
        // Store execution details for audit and monitoring
        executionRepository.save(execution);
    }
}
```

**ETL Execution Model:**
```java
@Entity
public class ETLExecution {
    @Id
    private String id = UUID.randomUUID().toString();
    
    private int extractedCount;
    private int transformedCount;
    private int loadedCount;
    private int verifiedCount;
    
    private String status; // SUCCESS, FAILED, PARTIAL
    private String errorMessage;
    
    private LocalDateTime startTime;
    private LocalDateTime completedTime;
    private Duration duration;
    
    @Lob
    private List<TransformedRecord> transformedData;
}
```

**Key Considerations:**
1. **Batch Processing**: Process data in batches to manage memory
2. **Error Handling**: Fail gracefully, log errors clearly
3. **Idempotency**: Upsert operations safe to re-run
4. **Monitoring**: Track metrics at each phase
5. **Data Quality**: Validate before each phase
6. **Performance**: Parallelize independent extractions
7. **Rollback**: Store state to resume on failure

---

#### Q8: Compare three approaches to handling 1 billion records: batch processing, streaming, and micro-batching.

**Answer:**

**Batch Processing:**

```
Processing Model:
Extract all data -> Transform all -> Load all

Characteristics:
- High latency (hours)
- High throughput (process all in one go)
- Scheduled (e.g., nightly)

Time: 6 hours to process 1B records
Latency: 6 hours from collection to availability

Implementation:
spark.read.parquet("s3://data")
    .repartition(1000)
    .map(transform)
    .write.parquet("s3://output")

Advantages:
✓ Resource efficient
✓ Handle complex transformations
✓ Easy error recovery (restart batch)
✓ Proven, stable technology
✓ Better compression (batch jobs)

Disadvantages:
✗ High latency not acceptable for real-time
✗ All-or-nothing: failure loses entire batch
✗ Resource utilization: peaks at batch time, idle otherwise

Use Cases:
- Nightly data warehouse updates
- Monthly billing calculations
- Daily ETL processes
- Analytics dashboards updated daily
```

**Streaming:**

```
Processing Model:
Process each record as it arrives

Characteristics:
- Very low latency (milliseconds/seconds)
- Lower per-record throughput
- Continuous

Time: Continuous processing
Latency: <1 second from arrival to availability

Implementation (Kafka + Spark):
kafkaStream.map(record -> transform(record))
    .sink(result -> sendToDatabase(result))

Advantages:
✓ Real-time insights
✓ Immediate anomaly detection
✓ Better resource utilization (constant)
✓ Quick user feedback

Disadvantages:
✗ Complex state management
✗ Exactly-once processing hard to guarantee
✗ Higher operational complexity
✗ Harder to handle late/out-of-order data
✗ More expensive (always-on clusters)

Use Cases:
- Stock price updates
- Fraud detection
- Real-time recommendations
- Live monitoring dashboards
- Sensor data processing
```

**Micro-batching:**

```
Processing Model:
Collect records for X seconds, process as batch

Characteristics:
- Medium latency (seconds/minutes)
- Medium throughput
- Periodic but frequent

Time: 5-minute batches, 2-5 minute latency
Latency: 5-10 minutes total

Implementation (Spark Streaming):
def processStream = new StreamingContext(sparkConf, Seconds(5))

stream.window(Minutes(5))
    .foreachRDD(rdd => process(rdd))

Advantages:
✓ Near real-time
✓ Leverage batch processing stability
✓ Good resource utilization
✓ Simpler state management
✓ Better fault tolerance
✓ Easier testing and debugging

Disadvantages:
✗ Not true real-time
✗ Still needs stream management
✗ Window management complexity
✗ Duplicate processing chance

Use Cases:
- Real-time analytics
- Dashboard updates every 5 minutes
- Alerting systems (5-minute windows)
- Load balancing
- Log aggregation
```

**Comparison Table:**

```
Aspect              | Batch          | Streaming      | Micro-batching
Latency             | Hours          | Seconds        | Seconds/Minutes
Throughput/Record   | Very High      | Medium         | High
Resource Util.      | Periodic spike | Constant       | Constant
Fault Tolerance     | Excellent      | Difficult      | Good
State Management    | Simple         | Complex        | Medium
Development Ease    | Easy           | Complex        | Medium
Setup Complexity    | Simple         | Complex        | Medium
Cost (1B records)   | $10-50         | $50-200        | $30-100
Exactly Once        | Built-in       | Hard           | Supported
Processing Delay    | 6 hours        | <1 sec         | 5-60 sec
Ordering            | No guarantee   | Hard to ensure | Window-based
```

**Choosing the Right Approach:**

```
Choose Batch If:
- Can wait hours for results
- Processing complex transformations
- High volume (billions of records)
- Cost is critical
- Example: Daily data warehouse load

Choose Streaming If:
- Need immediate results (<1 second)
- Low volume acceptable
- Budget available
- Can handle operational complexity
- Example: Stock price updates, fraud detection

Choose Micro-batching If:
- Need near real-time (5-30 minutes)
- High volume
- Want stability of batch with streaming speed
- Example: Real-time analytics, dashboards, monitoring
```

**Real-World Example: Processing 1B Records Daily**

```
Scenario 1: Batch
- 1B records arrive throughout day
- Run nightly ETL 10 PM - 4 AM
- Data available next morning for reporting
- Cost: $20/night

Scenario 2: Streaming
- Each record processed as it arrives <100ms
- Real-time dashboard updated continuously
- Always-on Kafka + Spark clusters
- Cost: $200/day

Scenario 3: Micro-batching (Recommended)
- Process in 5-minute windows
- 288 batches/day of ~3.5M records each
- Data available within 10 minutes
- Good balance of cost and latency
- Cost: $60/day
```

**Implementation Comparison:**

```java
// Batch Processing
SparkSession spark = SparkSession.builder().getOrCreate();
Dataset<Record> df = spark.read().parquet("s3://input");
df.repartition(1000)
    .map(record -> transform(record), Encoders.bean(Record.class))
    .write().mode("overwrite").parquet("s3://output");

// Streaming
SparkSession spark = SparkSession.builder().getOrCreate();
Dataset<Record> df = spark
    .readStream()
    .format("kafka")
    .option("kafka.bootstrap.servers", "localhost:9092")
    .option("subscribe", "topic")
    .load();
df.map(record -> transform(record), Encoders.bean(Record.class))
    .writeStream()
    .format("kafka")
    .option("kafka.bootstrap.servers", "localhost:9092")
    .option("topic", "output")
    .start();

// Micro-batching
SparkSession spark = SparkSession.builder().getOrCreate();
Dataset<Record> df = spark
    .readStream()
    .format("kafka")
    .option("subscribe", "topic")
    .load();
df.map(record -> transform(record), Encoders.bean(Record.class))
    .writeStream()
    .trigger(Trigger.ProcessingTime("5 minutes"))
    .option("checkpointLocation", "/tmp/checkpoint")
    .mode("append")
    .start();
```

---

#### Q9: How would you design a Java microservice that consumes from Kafka and writes to multiple data systems?

**Answer:**

**Architecture:**

```
Kafka Topic (source of truth)
    |
Java Microservice (Consumer + Router)
    |
    ├─> PostgreSQL (OLTP)
    ├─> Elasticsearch (Search)
    ├─> Redis (Cache)
    ├─> Data Lake (S3)
    └─> Analytics DB
        (Snowflake)
```

**Implementation:**

```java
@Configuration
@EnableKafka
public class KafkaConsumerConfig {
    
    @Bean
    public ConsumerFactory<String, CustomerEvent> consumerFactory() {
        Map<String, Object> configProps = new HashMap<>();
        configProps.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        configProps.put(ConsumerConfig.GROUP_ID_CONFIG, "customer-service");
        configProps.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        configProps.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, JsonDeserializer.class);
        configProps.put(JsonDeserializer.VALUE_DEFAULT_TYPE, CustomerEvent.class);
        configProps.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        configProps.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false); // Manual commit
        
        return new DefaultKafkaConsumerFactory<>(configProps);
    }
    
    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, CustomerEvent> 
        kafkaListenerContainerFactory() {
        
        ConcurrentKafkaListenerContainerFactory<String, CustomerEvent> factory =
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory());
        factory.setConcurrency(3); // 3 parallel consumers
        factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.MANUAL_IMMEDIATE);
        
        return factory;
    }
}

@Service
public class MultiDestinationKafkaConsumer {
    
    @Autowired
    private PostgreSQLService postgresService;
    
    @Autowired
    private ElasticsearchService elasticsearchService;
    
    @Autowired
    private RedisService redisService;
    
    @Autowired
    private DataLakeService dataLakeService;
    
    @Autowired
    private SnowflakeService snowflakeService;
    
    @Autowired
    private KafkaTemplate<String, CustomerEvent> kafkaTemplate;
    
    private static final Logger logger = LoggerFactory.getLogger(MultiDestinationKafkaConsumer.class);
    
    @KafkaListener(topics = "customer-events", groupId = "customer-service")
    public void consumeCustomerEvent(
            ConsumerRecord<String, CustomerEvent> record,
            Acknowledgment acknowledgment) {
        
        CustomerEvent event = record.value();
        String eventId = UUID.randomUUID().toString();
        
        logger.info("Received event: {} from partition: {} with offset: {}",
            event.getCustomerId(), record.getPartition(), record.getOffset());
        
        try {
            // Parallel write to multiple destinations
            CompletableFuture<Void> writeOperations = CompletableFuture.allOf(
                writeToPostgres(event, eventId),
                writeToElasticsearch(event, eventId),
                updateRedisCache(event, eventId),
                writeToDataLake(event, eventId)
            );
            
            writeOperations.join(); // Wait for all writes
            
            // Async write to analytics (non-blocking)
            writeToSnowflakeAsync(event, eventId);
            
            // Manual commit on success
            acknowledgment.acknowledge();
            logger.info("Successfully processed event: {}", eventId);
            
        } catch (Exception e) {
            logger.error("Error processing event: {}", eventId, e);
            // Send to DLQ for manual inspection
            sendToDeadLetterQueue(event, e);
            acknowledgment.acknowledge(); // Still commit to avoid reprocessing
        }
    }
    
    // PostgreSQL: Transactional, consistent data
    private CompletableFuture<Void> writeToPostgres(CustomerEvent event, String eventId) {
        return CompletableFuture.runAsync(() -> {
            try {
                Customer customer = new Customer();
                customer.setId(event.getCustomerId());
                customer.setName(event.getName());
                customer.setEmail(event.getEmail());
                customer.setEventId(eventId);
                customer.setProcessedAt(LocalDateTime.now());
                
                postgresService.saveOrUpdate(customer);
                logger.debug("Written to PostgreSQL: {}", event.getCustomerId());
                
            } catch (Exception e) {
                throw new CompletionException("PostgreSQL write failed", e);
            }
        });
    }
    
    // Elasticsearch: Full-text search capability
    private CompletableFuture<Void> writeToElasticsearch(CustomerEvent event, String eventId) {
        return CompletableFuture.runAsync(() -> {
            try {
                CustomerDocument doc = CustomerDocument.builder()
                    .id(event.getCustomerId())
                    .name(event.getName())
                    .email(event.getEmail())
                    .city(event.getCity())
                    .eventId(eventId)
                    .timestamp(LocalDateTime.now())
                    .build();
                
                elasticsearchService.indexDocument(doc);
                logger.debug("Indexed in Elasticsearch: {}", event.getCustomerId());
                
            } catch (Exception e) {
                throw new CompletionException("Elasticsearch write failed", e);
            }
        });
    }
    
    // Redis: Cache for quick lookup
    private CompletableFuture<Void> updateRedisCache(CustomerEvent event, String eventId) {
        return CompletableFuture.runAsync(() -> {
            try {
                CustomerCache cache = CustomerCache.builder()
                    .id(event.getCustomerId())
                    .name(event.getName())
                    .email(event.getEmail())
                    .ttl(3600) // 1 hour
                    .build();
                
                redisService.setWithExpiry("customer:" + event.getCustomerId(), cache, 3600);
                logger.debug("Cached in Redis: {}", event.getCustomerId());
                
            } catch (Exception e) {
                logger.warn("Redis cache update failed (non-critical): {}", eventId);
                // Don't throw, Redis failure shouldn't block other writes
            }
        });
    }
    
    // Data Lake: Raw data archival
    private CompletableFuture<Void> writeToDataLake(CustomerEvent event, String eventId) {
        return CompletableFuture.runAsync(() -> {
            try {
                DataRecord record = DataRecord.builder()
                    .id(event.getCustomerId())
                    .eventType(event.getEventType())
                    .payload(event)
                    .eventId(eventId)
                    .timestamp(LocalDateTime.now())
                    .build();
                
                String s3Path = String.format("s3://data-lake/customer-events/%s/%s/",
                    LocalDate.now(), LocalTime.now().getHour());
                
                dataLakeService.writeToS3(s3Path, record);
                logger.debug("Written to Data Lake: {}", eventId);
                
            } catch (Exception e) {
                throw new CompletionException("Data Lake write failed", e);
            }
        });
    }
    
    // Snowflake: Analytics (fire-and-forget, non-blocking)
    private void writeToSnowflakeAsync(CustomerEvent event, String eventId) {
        CompletableFuture.runAsync(() -> {
            try {
                AnalyticsEvent analyticsEvent = AnalyticsEvent.builder()
                    .customerId(event.getCustomerId())
                    .eventType(event.getEventType())
                    .eventId(eventId)
                    .timestamp(LocalDateTime.now())
                    .build();
                
                snowflakeService.insertEvent(analyticsEvent);
                logger.debug("Inserted into Snowflake: {}", eventId);
                
            } catch (Exception e) {
                logger.error("Snowflake insert failed (async): {}", eventId, e);
                // Snowflake failure logged but doesn't block consumer
            }
        });
    }
    
    // Dead Letter Queue: Handle poison messages
    private void sendToDeadLetterQueue(CustomerEvent event, Exception error) {
        try {
            DeadLetterMessage dlq = DeadLetterMessage.builder()
                .originalEvent(event)
                .errorMessage(error.getMessage())
                .errorStacktrace(getStackTrace(error))
                .receivedAt(LocalDateTime.now())
                .build();
            
            kafkaTemplate.send("customer-events-dlq", event.getCustomerId(), dlq);
            logger.error("Event sent to DLQ: {}", event.getCustomerId());
            
        } catch (Exception e) {
            logger.error("Failed to send to DLQ", e);
            // Send alert to ops team
        }
    }
}

// Service interfaces
@Service
public class PostgreSQLService {
    @Autowired
    private CustomerRepository repository;
    
    @Transactional
    public void saveOrUpdate(Customer customer) {
        // Check if exists for idempotency
        Optional<Customer> existing = repository.findById(customer.getId());
        if (existing.isPresent() && 
            existing.get().getEventId().equals(customer.getEventId())) {
            return; // Already processed
        }
        repository.save(customer);
    }
}

@Service
public class ElasticsearchService {
    @Autowired
    private RestHighLevelClient client;
    
    public void indexDocument(CustomerDocument doc) throws IOException {
        IndexRequest request = new IndexRequest("customers")
            .id(doc.getId())
            .source(convertToMap(doc), XContentType.JSON);
        
        client.index(request, RequestOptions.DEFAULT);
    }
}

@Service
public class RedisService {
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    public void setWithExpiry(String key, Object value, long expirySeconds) {
        redisTemplate.opsForValue().set(key, value, expirySeconds, TimeUnit.SECONDS);
    }
}

@Service
public class DataLakeService {
    @Autowired
    private AmazonS3 s3Client;
    
    public void writeToS3(String path, DataRecord record) {
        // Write in Parquet format
        byte[] data = convertToParquet(record);
        s3Client.putObject(new PutObjectRequest(
            BUCKET_NAME, path + UUID.randomUUID() + ".parquet",
            new ByteArrayInputStream(data), new ObjectMetadata()
        ));
    }
}

@Service
public class SnowflakeService {
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    @Async
    public void insertEvent(AnalyticsEvent event) {
        String sql = "INSERT INTO analytics_events (customer_id, event_type, timestamp) " +
                    "VALUES (?, ?, ?)";
        jdbcTemplate.update(sql, event.getCustomerId(), event.getEventType(), 
            Timestamp.valueOf(event.getTimestamp()));
    }
}
```

**Error Handling & Resilience:**

```java
@Configuration
public class ResiliencePatterns {
    
    // Retry policy
    @Bean
    public RetryTemplate retryTemplate() {
        RetryTemplate template = new RetryTemplate();
        
        FixedBackOffPolicy backOffPolicy = new FixedBackOffPolicy();
        backOffPolicy.setBackOffPeriod(1000); // 1 second
        template.setBackOffPolicy(backOffPolicy);
        
        SimpleRetryPolicy retryPolicy = new SimpleRetryPolicy();
        retryPolicy.setMaxAttempts(3);
        template.setRetryPolicy(retryPolicy);
        
        return template;
    }
    
    // Circuit breaker
    @Bean
    public CircuitBreakerFactory circuitBreakerFactory() {
        return new Resilience4JCircuitBreakerFactory();
    }
}
```

**Key Design Patterns:**
1. **Fanout**: Write to multiple destinations
2. **Idempotency**: Same message processed multiple times = same result
3. **Async writes**: Non-critical destinations don't block
4. **DLQ**: Failed messages sent for manual review
5. **Manual offsets**: Better error handling control

---

#### Q10: Design a data synchronization system between a transactional database and a data warehouse.

**Answer:**

**System Architecture:**

```
Transactional Database (OLTP)
    |
    |-> Change Data Capture (CDC)
    |
Message Queue (Kafka)
    |
    |-> Transformation Engine
    |
Data Warehouse (OLAP) with SCD Type 2
    |
    |-> Reporting/Analytics
```

**Complete Implementation:**

```java
// Step 1: CDC Setup using Debezium concept
@Service
public class ChangeDataCaptureService {
    
    @Autowired
    private DataSourceProperties dataSourceProperties;
    
    @Autowired
    private KafkaTemplate<String, ChangeEvent> kafkaTemplate;
    
    /**
     * Setup CDC using transaction logs
     * For Oracle: Use LogMiner
     * For PostgreSQL: Use logical decoding
     * For MySQL: Use binary logs
     */
    public void setupCDC() {
        // This would be done via Debezium connector in production
        // Here's Java representation of what happens:
        /*
        Debezium Connector Configuration:
        {
          "name": "oracle-cdc-connector",
          "config": {
            "connector.class": "io.debezium.connector.oracle.OracleConnector",
            "database.hostname": "localhost",
            "database.port": 1521,
            "database.user": "dbuser",
            "database.dbname": "ORCL",
            "database.history.kafka.bootstrap.servers": "localhost:9092",
            "database.history.kafka.topic": "schema-changes",
            "table.include.list": "customers,orders,products",
            "topic.prefix": "cdc"
          }
        }
        */
    }
}

// Step 2: CDC Message consumption and routing
@Service
public class CDCMessageConsumer {
    
    @Autowired
    private DimensionTransformer dimensionTransformer;
    
    @Autowired  
    private FactTransformer factTransformer;
    
    @Autowired
    private KafkaTemplate<String, TransformedMessage> kafkaTemplate;
    
    @KafkaListener(topics = "cdc.customers,cdc.orders,cdc.products", groupId = "dw-sync")
    public void consumeCDCEvent(ConsumerRecord<String, ChangeEvent> record) {
        ChangeEvent event = record.value();
        
        logger.info("CDC Event - Table: {}, Op: {}, Id: {}",
            event.getTable(), event.getOperation(), event.getEntityId());
        
        try {
            // Route based on table
            switch (event.getTable()) {
                case "CUSTOMERS":
                    processDimensionChange(event, "CUSTOMER_DIM");
                    break;
                case "PRODUCTS":
                    processDimensionChange(event, "PRODUCT_DIM");
                    break;
                case "ORDERS":
                    processFactChange(event);
                    break;
                case "ORDER_ITEMS":
                    processFactChange(event);
                    break;
                default:
                    logger.warn("Unknown table: {}", event.getTable());
            }
        } catch (Exception e) {
            logger.error("Failed to process CDC event", e);
            sendToErrorTopic(event, e);
        }
    }
    
    // Dimension processing with SCD Type 2
    private void processDimensionChange(ChangeEvent event, String dimensionName) {
        TransformedMessage message = TransformedMessage.builder()
            .id(UUID.randomUUID().toString())
            .sourceTable(event.getTable())
            .targetDimension(dimensionName)
            .operation(event.getOperation())
            .beforeData(event.getBefore())
            .afterData(event.getAfter())
            .timestamp(LocalDateTime.now())
            .scdType(2) // Track historical changes
            .build();
        
        // Send to dimension transformation topic
        kafkaTemplate.send(dimensionName + "-transform", message);
    }
    
    // Fact processing
    private void processFactChange(ChangeEvent event) {
        TransformedMessage message = TransformedMessage.builder()
            .id(UUID.randomUUID().toString())
            .sourceTable(event.getTable())
            .operation(event.getOperation())
            .afterData(event.getAfter())
            .timestamp(LocalDateTime.now())
            .build();
        
        kafkaTemplate.send("FACT_TRANSFORM", message);
    }
}

// Step 3: Model classes
@Data
@Builder
public class ChangeEvent {
    private String eventId;
    private String table;
    private String operation; // INSERT, UPDATE, DELETE
    private Long entityId;
    private Map<String, Object> before; // Old values for UPDATE
    private Map<String, Object> after;  // New values
    private LocalDateTime eventTimestamp;
}

@Data
@Builder
public class TransformedMessage {
    private String id;
    private String sourceTable;
    private String targetDimension;
    private String operation;
    private Map<String, Object> beforeData;
    private Map<String, Object> afterData;
    private LocalDateTime timestamp;
    private Integer scdType;
}

// Step 4: Dimension transformation with SCD Type 2
@Service
public class DimensionTransformer {
    
    @Autowired
    private DataWarehouseRepository dwRepository;
    
    @Autowired
    private KafkaTemplate<String, DimensionLoadEvent> kafkaTemplate;
    
    @KafkaListener(topics = "CUSTOMER_DIM-transform")
    public void transformAndLoadDimension(TransformedMessage message) {
        try {
            DimensionRecord record = transformToDimension(message);
            
            // Before loading, check if this is a change
            DimensionRecord existing = dwRepository.findCurrentDimension(
                record.getBusinessKey());
            
            if (existing == null) {
                // New record
                loadNewDimension(record);
            } else if (hasRelevantChange(existing, record)) {
                // SCD Type 2: Close current record, add new
                closeExistingDimension(existing);
                loadNewDimension(record);
            } else {
                // No relevant change (e.g., only technical fields)
                updateCurrentDimension(existing, record);
            }
            
        } catch (Exception e) {
            logger.error("Dimension transformation failed", e);
            throw new SyncException("Failed to transform dimension", e);
        }
    }
    
    private void loadNewDimension(DimensionRecord record) {
        record.setDimensionKey(generateDimensionKey());
        record.setEffectiveFrom(LocalDate.now());
        record.setEffectiveTo(LocalDate.of(9999, 12, 31));
        record.setIsCurrent(true);
        
        dwRepository.saveDimension(record);
        logger.info("Loaded new dimension: {}", record.getBusinessKey());
    }
    
    private void closeExistingDimension(DimensionRecord existing) {
        existing.setEffectiveTo(LocalDate.now().minusDays(1));
        existing.setIsCurrent(false);
        dwRepository.updateDimension(existing);
    }
    
    private void updateCurrentDimension(DimensionRecord existing, DimensionRecord updated) {
        existing.setName(updated.getName());
        existing.setAddress(updated.getAddress());
        existing.setLastUpdated(LocalDateTime.now());
        dwRepository.updateDimension(existing);
    }
    
    private boolean hasRelevantChange(DimensionRecord old, DimensionRecord new_) {
        // Check business-relevant fields only
        return !Objects.equals(old.getName(), new_.getName()) ||
               !Objects.equals(old.getAddress(), new_.getAddress()) ||
               !Objects.equals(old.getStatus(), new_.getStatus());
    }
    
    private DimensionRecord transformToDimension(TransformedMessage message) {
        Map<String, Object> data = message.getAfterData();
        
        return DimensionRecord.builder()
            .businessKey(data.get("CUSTOMER_ID").toString())
            .name((String) data.get("NAME"))
            .email((String) data.get("EMAIL"))
            .address((String) data.get("ADDRESS"))
            .status((String) data.get("STATUS"))
            .sourceSystem("OLTP")
            .lastUpdated(LocalDateTime.now())
            .build();
    }
}

// Step 5: Fact transformation
@Service
public class FactTransformer {
    
    @Autowired
    private DataWarehouseRepository dwRepository;
    
    @KafkaListener(topics = "FACT_TRANSFORM")
    public void transformAndLoadFact(TransformedMessage message) {
        try {
            FactRecord fact = transformToFact(message);
            
            // For facts, use upsert for idempotency
            dwRepository.upsertFact(fact);
            logger.info("Loaded fact: {}", fact.getFactKey());
            
        } catch (Exception e) {
            logger.error("Fact transformation failed", e);
            throw new SyncException("Failed to transform fact", e);
        }
    }
    
    private FactRecord transformToFact(TransformedMessage message) {
        Map<String, Object> data = message.getAfterData();
        
        // Lookup dimension keys
        String customerDimKey = dwRepository.lookupDimensionKey(
            "CUSTOMER_DIM", data.get("CUSTOMER_ID").toString());
        
        String productDimKey = dwRepository.lookupDimensionKey(
            "PRODUCT_DIM", data.get("PRODUCT_ID").toString());
        
        String dateDimKey = dwRepository.lookupDateDimension(
            (LocalDate) data.get("ORDER_DATE"));
        
        return FactRecord.builder()
            .factKey(generateFactKey())
            .customerDimKey(customerDimKey)
            .productDimKey(productDimKey)
            .dateDimKey(dateDimKey)
            .amount(new BigDecimal(data.get("AMOUNT").toString()))
            .quantity(Integer.parseInt(data.get("QUANTITY").toString()))
            .sourceId(data.get("ORDER_ID").toString())
            .sourceSystem("OLTP")
            .loadedAt(LocalDateTime.now())
            .build();
    }
}

// Step 6: Data Warehouse Repository
@Repository
public class DataWarehouseRepository {
    
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    public void saveDimension(DimensionRecord record) {
        String sql = """
            INSERT INTO CUSTOMER_DIM (
                DIM_KEY, BUSINESS_KEY, NAME, EMAIL, ADDRESS, STATUS,
                EFFECTIVE_FROM, EFFECTIVE_TO, IS_CURRENT, LOADED_AT
            ) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
            """;
        
        jdbcTemplate.update(sql,
            record.getDimensionKey(),
            record.getBusinessKey(),
            record.getName(),
            record.getEmail(),
            record.getAddress(),
            record.getStatus(),
            Date.valueOf(record.getEffectiveFrom()),
            Date.valueOf(record.getEffectiveTo()),
            record.getIsCurrent(),
            Timestamp.valueOf(record.getLastUpdated())
        );
    }
    
    public void upsertFact(FactRecord fact) {
        String sql = """
            MERGE INTO ORDER_FACT f
            USING (SELECT ? as SOURCE_ID) s
            ON f.SOURCE_ID = s.SOURCE_ID
            WHEN MATCHED THEN
                UPDATE SET amount = ?, quantity = ?, loaded_at = ?
            WHEN NOT MATCHED THEN
                INSERT (FACT_KEY, CUSTOMER_DIM_KEY, PRODUCT_DIM_KEY, DATE_DIM_KEY,
                        AMOUNT, QUANTITY, SOURCE_ID, LOADED_AT)
                VALUES (?, ?, ?, ?, ?, ?, ?, ?)
            """;
        
        jdbcTemplate.update(sql,
            fact.getSourceId(), // Merge condition
            fact.getAmount(), fact.getQuantity(), Timestamp.valueOf(fact.getLoadedAt()),
            // Insert values
            fact.getFactKey(), fact.getCustomerDimKey(), fact.getProductDimKey(),
            fact.getDateDimKey(), fact.getAmount(), fact.getQuantity(),
            fact.getSourceId(), Timestamp.valueOf(fact.getLoadedAt())
        );
    }
    
    public DimensionRecord findCurrentDimension(String businessKey) {
        String sql = """
            SELECT * FROM CUSTOMER_DIM
            WHERE BUSINESS_KEY = ? AND IS_CURRENT = true
            """;
        
        List<DimensionRecord> results = jdbcTemplate.query(sql,
            new DimensionRowMapper(), businessKey);
        
        return results.isEmpty() ? null : results.get(0);
    }
}

// Step 7: Monitoring and Reconciliation
@Service
public class SyncMonitoringService {
    
    @Autowired
    private DataWarehouseRepository dwRepository;
    
    @Autowired
    private TransactionalRepository transactionalRepo;
    
    @Scheduled(fixedRate = 3600000) // Hourly reconciliation
    public void reconcileData() {
        try {
            logger.info("Starting data reconciliation...");
            
            // Check record counts
            long olpCount = transactionalRepo.countCustomers();
            long dwCount = dwRepository.countCurrentDimensions("CUSTOMER_DIM");
            
            if (olpCount != dwCount) {
                logger.warn("Record count mismatch! OLTP: {}, DW: {}", olpCount, dwCount);
                sendAlert("Data Sync Discrepancy", 
                    "OLTP has " + olpCount + " records, DW has " + dwCount);
            }
            
            // Check for late arrivals
            List<String> missingInDW = findMissingRecords();
            if (!missingInDW.isEmpty()) {
                logger.warn("Found {} records in OLTP but not in DW", missingInDW.size());
                // Trigger manual reconciliation
            }
            
            // Data quality checks
            validateDataQuality();
            
            logger.info("Reconciliation completed");
            
        } catch (Exception e) {
            logger.error("Reconciliation failed", e);
        }
    }
    
    private List<String> findMissingRecords() {
        String sql = """
            SELECT c.CUSTOMER_ID FROM CUSTOMERS c
            WHERE NOT EXISTS (
                SELECT 1 FROM CUSTOMER_DIM d
                WHERE d.BUSINESS_KEY = c.CUSTOMER_ID
                AND d.IS_CURRENT = true
            )
            """;
        
        return transactionalRepo.findMissingIds(sql);
    }
    
    private void validateDataQuality() {
        // Check for NULL values
        // Check data type mismatches
        // Check for outliers/anomalies
    }
}

// Configuration
@Configuration
public class DataSyncConfiguration {
    
    @Bean
    public TaskScheduler taskScheduler() {
        return new ThreadPoolTaskScheduler();
    }
}
```

**Schema Design:**

```sql
-- Dimension Table with SCD Type 2
CREATE TABLE CUSTOMER_DIM (
    DIM_KEY BIGINT PRIMARY KEY,
    BUSINESS_KEY VARCHAR(50) NOT NULL,
    NAME VARCHAR(100),
    EMAIL VARCHAR(100),
    ADDRESS VARCHAR(200),
    STATUS VARCHAR(20),
    EFFECTIVE_FROM DATE NOT NULL,
    EFFECTIVE_TO DATE NOT NULL,
    IS_CURRENT BOOLEAN NOT NULL,
    LOADED_AT TIMESTAMP,
    INDEX (BUSINESS_KEY, IS_CURRENT)
);

-- Fact Table
CREATE TABLE ORDER_FACT (
    FACT_KEY BIGINT PRIMARY KEY,
    CUSTOMER_DIM_KEY BIGINT NOT NULL,
    PRODUCT_DIM_KEY BIGINT NOT NULL,
    DATE_DIM_KEY INT NOT NULL,
    AMOUNT DECIMAL(10,2),
    QUANTITY INT,
    SOURCE_ID VARCHAR(50) NOT NULL UNIQUE,
    LOADED_AT TIMESTAMP,
    FOREIGN KEY (CUSTOMER_DIM_KEY) REFERENCES CUSTOMER_DIM(DIM_KEY),
    INDEX (SOURCE_ID)
);
```

**Key Features:**
1. **CDC**: Captures changes in real-time
2. **SCD Type 2**: Maintains history
3. **Idempotency**: Safe to reprocess
4. **Reconciliation**: Hourly checks
5. **Error Handling**: DLQ for failures
6. **Monitoring**: Latency and data quality metrics

---

## Key Concepts Deep Dive

[Previous sections covered above in detail]

---

## Java Implementation Examples

### Connection Pooling with HikariCP
```java
@Configuration
public class DataSourceConfiguration {
    
    @Bean
    public DataSource dataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:postgresql://localhost:5432/mydb");
        config.setUsername("user");
        config.setPassword("password");
        config.setMaximumPoolSize(20);
        config.setMinimumIdle(5);
        config.setConnectionTimeout(30000);
        config.setIdleTimeout(600000);
        config.setMaxLifetime(1800000);
        config.setAutoCommit(true);
        
        return new HikariDataSource(config);
    }
}
```

### Batch Operations
```java
@Service
public class BatchInsertService {
    
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    public void batchInsertRecords(List<DataRecord> records) {
        String sql = "INSERT INTO data_records (id, name, value) VALUES (?, ?, ?)";
        
        int[] result = jdbcTemplate.batchUpdate(sql, new BatchPreparedStatementSetter() {
            @Override
            public void setValues(PreparedStatement ps, int i) throws SQLException {
                DataRecord record = records.get(i);
                ps.setLong(1, record.getId());
                ps.setString(2, record.getName());
                ps.setString(3, record.getValue());
            }
            
            @Override
            public int getBatchSize() {
                return records.size();
            }
        });
        
        logger.info("Batch insert completed. Updated records: {}", 
            Arrays.stream(result).sum());
    }
}
```

---

### Topic 9: Java-Specific Data Integration

**What You Need to Know:**

**ORM Frameworks Overview:**

```
Hibernate vs JPA vs MyBatis:

Hibernate:
- Full ORM implementation
- Automatic state management
- Query caching
- Good for complex object graphs
- Learning curve steeper

JPA (Java Persistence API):
- Standard specification
- Can use different implementations (Hibernate, EclipseLink)
- Standardized across Java ecosystem
- Recommended for enterprise apps

MyBatis:
- Not a full ORM, semi-automatic
- Direct SQL control
- Good for complex queries
- Faster development for SQL experts
- Less magic, more control
```

**Lazy vs Eager Loading:**

```java
// Problem: Lazy Loading
@Entity
public class Order {
    @ManyToOne(fetch = FetchType.LAZY)
    private Customer customer;
}

Order order = repository.findById(1L);
String customerName = order.getCustomer().getName(); // SECOND query here!

// Solution 1: Eager Load
@Entity
public class Order {
    @ManyToOne(fetch = FetchType.EAGER)
    private Customer customer;
}

// Solution 2: Explicit Join Fetch
@Query("SELECT o FROM Order o JOIN FETCH o.customer WHERE o.id = :id")
Order findByIdWithCustomer(@Param("id") Long id);

// Solution 3: Entity Graph
@EntityGraph(attributePaths = "customer")
@Query("SELECT o FROM Order o WHERE o.id = :id")
Order findByIdWithGraph(@Param("id") Long id);
```

**Connection Management in Java:**

```java
@Configuration
public class JDBCConnectionConfig {
    
    // Direct JDBC (Low-level)
    public void simpleJDBCExample() throws SQLException {
        String url = "jdbc:postgresql://localhost:5432/mydb";
        String user = "admin";
        String password = "password";
        
        Connection conn = DriverManager.getConnection(url, user, password);
        Statement stmt = conn.createStatement();
        ResultSet rs = stmt.executeQuery("SELECT * FROM users");
        
        while (rs.next()) {
            System.out.println(rs.getString("name"));
        }
        
        rs.close();
        stmt.close();
        conn.close();
    }
    
    // With DataSource (Better)
    @Bean
    public DataSource dataSource() {
        DriverManagerDataSource dataSource = new DriverManagerDataSource();
        dataSource.setDriverClassName("org.postgresql.Driver");
        dataSource.setUrl("jdbc:postgresql://localhost:5432/mydb");
        dataSource.setUsername("admin");
        dataSource.setPassword("password");
        return dataSource;
    }
    
    // With Connection Pool (Best)
    @Bean
    public DataSource connectionPooledDataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:postgresql://localhost:5432/mydb");
        config.setUsername("admin");
        config.setPassword("password");
        config.setMaximumPoolSize(20);
        config.setMinimumIdle(5);
        return new HikariDataSource(config);
    }
}
```

**Handling Large Result Sets:**

```java
@Service
public class LargeResultSetService {
    
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    // Bad: Load all in memory
    public void processBadly() {
        List<User> users = jdbcTemplate.query(
            "SELECT * FROM users",
            new UserRowMapper()
        ); // Can cause OutOfMemoryException for millions of rows
        
        users.forEach(user -> processUser(user));
    }
    
    // Good: Stream processing
    public void processGood() {
        String sql = "SELECT * FROM users WHERE status = 'ACTIVE'";
        
        jdbcTemplate.query(sql, rs -> {
            while (rs.next()) {
                User user = new User(
                    rs.getLong("id"),
                    rs.getString("name"),
                    rs.getString("email")
                );
                processUser(user); // Process one at a time
            }
        });
    }
    
    // Better: ResultSet streaming with pagination
    public void streamWithPagination(int pageSize) {
        int offset = 0;
        boolean hasMore = true;
        
        while (hasMore) {
            String sql = "SELECT * FROM users LIMIT ? OFFSET ?";
            List<User> batch = jdbcTemplate.query(
                sql,
                new Object[]{pageSize, offset},
                new UserRowMapper()
            );
            
            if (batch.isEmpty()) {
                hasMore = false;
            } else {
                batch.forEach(this::processUser);
                offset += pageSize;
            }
        }
    }
    
    // Best: Spring Data streaming
    public void streamWithRepository() {
        userRepository.findAllAsStream()
            .forEach(this::processUser);
    }
}
```

**Spring Data and Repository Pattern:**

```java
// Repository Interface
public interface UserRepository extends JpaRepository<User, Long> {
    
    // Simple finder methods (Spring Data auto-generates SQL)
    List<User> findByFirstName(String firstName);
    
    User findByEmail(String email);
    
    @Query("SELECT u FROM User u WHERE u.status = :status AND u.createdDate > :date")
    List<User> findActiveUsersAfterDate(
        @Param("status") String status,
        @Param("date") LocalDate date
    );
    
    @Query(value = "SELECT * FROM users WHERE LOWER(first_name) LIKE LOWER(CONCAT('%', :name, '%'))", 
           nativeQuery = true)
    List<User> searchByName(@Param("name") String name);
    
    Page<User> findByStatus(String status, Pageable pageable);
    
    Stream<User> findAllAsStream();
}

// Service
@Service
public class UserService {
    
    @Autowired
    private UserRepository userRepository;
    
    public List<User> findActiveUsers() {
        return userRepository.findActiveUsersAfterDate("ACTIVE", LocalDate.now().minusDays(30));
    }
    
    public Page<User> getPaginatedUsers(int page, int size) {
        Pageable pageable = PageRequest.of(page, size, Sort.by("createdDate").descending());
        return userRepository.findByStatus("ACTIVE", pageable);
    }
    
    @Transactional
    public void bulkUpdateStatus(List<Long> userIds, String newStatus) {
        userRepository.saveAll(
            userRepository.findAllById(userIds).stream()
                .peek(u -> u.setStatus(newStatus))
                .collect(Collectors.toList())
        );
    }
}
```

**Entity Relationships and Cascade Options:**

```java
@Entity
public class Author {
    @Id
    private Long id;
    private String name;
    
    // One-to-Many : Cascade operations
    @OneToMany(mappedBy = "author", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Book> books = new ArrayList<>();
}

@Entity
public class Book {
    @Id
    private Long id;
    private String title;
    
    @ManyToOne
    @JoinColumn(name = "author_id")
    private Author author;
    
    @ManyToMany
    @JoinTable(
        name = "book_category",
        joinColumns = @JoinColumn(name = "book_id"),
        inverseJoinColumns = @JoinColumn(name = "category_id")
    )
    private Set<Category> categories = new HashSet<>();
}

// Cascade behaviors:
// - PERSIST: Auto-save child when parent saved
// - MERGE: Auto-merge child when parent merged
// - REMOVE: Auto-delete child when parent deleted
// - ALL: All of above
// - orphanRemoval: Delete child if removed from parent collection
```

**Transaction Management:**

```java
@Configuration
@EnableTransactionManagement
public class TransactionConfig {
    
    @Bean
    public PlatformTransactionManager transactionManager(EntityManagerFactory emf) {
        return new JpaTransactionManager(emf);
    }
}

@Service
public class OrderService {
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Autowired
    private InventoryService inventoryService;
    
    // Declarative Transaction Management (Recommended)
    @Transactional
    public Order createOrder(OrderRequest request) {
        Order order = new Order();
        order.setCustomerId(request.getCustomerId());
        order.setAmount(request.getAmount());
        
        // Both operations in same transaction
        Order saved = orderRepository.save(order);
        inventoryService.decreaseStock(request.getProductId(), request.getQuantity());
        
        return saved;
    }
    
    // Programmatic Transaction Management (When needed)
    @Autowired
    private TransactionTemplate transactionTemplate;
    
    public Order createOrderProgrammatically(OrderRequest request) {
        return transactionTemplate.execute(status -> {
            Order order = new Order();
            order.setCustomerId(request.getCustomerId());
            
            Order saved = orderRepository.save(order);
            inventoryService.decreaseStock(request.getProductId(), request.getQuantity());
            
            return saved;
        });
    }
    
    // Isolation Levels
    // DEFAULT: Use database default
    // READ_UNCOMMITTED: Lowest isolation, highest performance
    // READ_COMMITTED: Standard
    // REPEATABLE_READ: Prevents dirty/non-repeatable reads
    // SERIALIZABLE: Highest isolation, lowest performance
    
    @Transactional(isolation = Isolation.REPEATABLE_READ, timeout = 30)
    public void criticalOperation() {
        // Critical business logic
    }
}
```

---

### Topic 10: Scalability & Architecture

**Sharding Strategies:**

```
Sharding: Partition data across multiple databases

Sharding Keys:
1. Range-based: ID ranges (1-1M, 1M-2M)
   - Pro: Simple, easy to implement
   - Con: Hotspots, uneven distribution

2. Hash-based: Hash(customer_id) % number_of_shards
   - Pro: Even distribution
   - Con: Requires rebalancing on shard addition

3. Directory-based: Lookup table maps key to shard
   - Pro: Flexible, handles rebalancing
   - Con: Adds extra lookup

Example:
Customers 1-1M -> Shard 1
Customers 1M-2M -> Shard 2
Customers 2M-3M -> Shard 3
```

**Distributed Transactions - Saga Pattern:**

```java
@Service
public class PaymentSaga {
    
    @Autowired
    private OrderService orderService;
    
    @Autowired
    private PaymentService paymentService;
    
    @Autowired
    private SagaOrchestratorService orchestrator;
    
    /**
     * Choreography Pattern: Services call each other
     */
    public void processOrderChoreography(Order order) {
        // Step 1: Create order
        Order created = orderService.createOrder(order);
        
        try {
            // Step 2: Process payment
            Payment payment = new Payment();
            payment.setOrderId(created.getId());
            payment.setAmount(created.getTotalAmount());
            paymentService.processPayment(payment);
            
            // Step 3: Ship order
            orderService.shipOrder(created.getId());
            
        } catch (PaymentFailedException e) {
            // Compensate: Cancel order
            orderService.cancelOrder(created.getId());
            throw e;
        }
    }
    
    /**
     * Orchestration Pattern: Central service coordinates
     */
    @Transactional
    public void processOrderOrchestration(Order order) {
        SagaExecution execution = new SagaExecution();
        execution.setOrderId(order.getId());
        
        try {
            // Step 1: Create order
            Order created = orderService.createOrder(order);
            execution.recordStep("order_created", created.getId());
            
            // Step 2: Process payment
            Payment payment = new Payment();
            payment.setOrderId(created.getId());
            payment.setAmount(created.getTotalAmount());
            paymentService.processPayment(payment);
            execution.recordStep("payment_processed", payment.getId());
            
            // Step 3: Ship order
            orderService.shipOrder(created.getId());
            execution.recordStep("order_shipped", created.getId());
            
            execution.setStatus("COMPLETED");
            
        } catch (Exception e) {
            execution.setStatus("FAILED");
            executeCompensation(execution);
        }
        
        orchestrator.saveExecution(execution);
    }
    
    private void executeCompensation(SagaExecution execution) {
        // Rollback in reverse order
        if (execution.hasStep("order_shipped")) {
            orderService.cancelShip(execution.getOrderId());
        }
        
        if (execution.hasStep("payment_processed")) {
            paymentService.refundPayment(execution.getOrderId());
        }
        
        if (execution.hasStep("order_created")) {
            orderService.cancelOrder(execution.getOrderId());
        }
    }
}
```

**API Design for Data Consumption:**

```java
@RestController
@RequestMapping("/api/orders")
public class OrderAPI {
    
    @Autowired
    private OrderService orderService;
    
    // Filtering
    @GetMapping
    public ResponseEntity<List<OrderDTO>> getOrders(
            @RequestParam(required = false) String status,
            @RequestParam(required = false) BigDecimal minAmount,
            @RequestParam(required = false) BigDecimal maxAmount,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        
        OrderFilter filter = OrderFilter.builder()
            .status(status)
            .minAmount(minAmount)
            .maxAmount(maxAmount)
            .build();
        
        Page<Order> orders = orderService.findOrders(filter, PageRequest.of(page, size));
        return ResponseEntity.ok(orders.map(this::convertToDTO).getContent());
    }
    
    // Sorting
    @GetMapping
    public ResponseEntity<List<OrderDTO>> getSortedOrders(
            @RequestParam(defaultValue = "createdAt,desc") String sort,
            @RequestParam(defaultValue = "0") int page) {
        
        String[] sortParts = sort.split(",");
        Sort.Direction direction = sortParts[1].equalsIgnoreCase("asc") 
            ? Sort.Direction.ASC : Sort.Direction.DESC;
        
        Pageable pageable = PageRequest.of(page, 20, Sort.by(direction, sortParts[0]));
        Page<Order> orders = orderService.findAllOrders(pageable);
        return ResponseEntity.ok(orders.map(this::convertToDTO).getContent());
    }
    
    // Aggregation
    @GetMapping("/summary")
    public ResponseEntity<OrderSummary> getOrderSummary(
            @RequestParam(required = false) LocalDate startDate,
            @RequestParam(required = false) LocalDate endDate) {
        
        OrderSummary summary = orderService.getOrderSummary(startDate, endDate);
        return ResponseEntity.ok(summary);
    }
    
    // Sparse fieldsets (return only requested fields)
    @GetMapping("/{id}")
    public ResponseEntity<OrderDTO> getOrder(
            @PathVariable Long id,
            @RequestParam(required = false) String fields) {
        
        Order order = orderService.findById(id);
        OrderDTO dto = convertToDTO(order);
        
        if (fields != null) {
            dto = filterFields(dto, fields.split(","));
        }
        
        return ResponseEntity.ok(dto);
    }
}

@Data
@Builder
public class OrderDTO {
    private Long id;
    private String customerId;
    private BigDecimal amount;
    private String status;
    private List<OrderItemDTO> items;
    private LocalDateTime createdAt;
}

@Data
@Builder
public class OrderSummary {
    private int totalOrders;
    private BigDecimal totalAmount;
    private BigDecimal averageAmount;
    private Map<String, Integer> ordersByStatus;
}
```

**Rate Limiting:**

```java
@Component
public class RateLimiterInterceptor implements HandlerInterceptor {
    
    @Autowired
    private RateLimitService rateLimitService;
    
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response,
                            Object handler) throws Exception {
        
        String clientId = getClientId(request);
        String endpoint = request.getRequestURI();
        
        if (!rateLimitService.isAllowed(clientId, endpoint)) {
            response.setStatus(HttpStatus.TOO_MANY_REQUESTS.value());
            response.setHeader("Retry-After", "60");
            response.getWriter().write("Rate limit exceeded");
            return false;
        }
        
        response.setHeader("X-RateLimit-Limit", "100");
        response.setHeader("X-RateLimit-Remaining", 
            String.valueOf(rateLimitService.getRemainingRequests(clientId, endpoint)));
        
        return true;
    }
}

@Service
public class RateLimitService {
    
    @Autowired
    private RedisTemplate<String, Integer> redisTemplate;
    
    private static final int MAX_REQUESTS = 100;
    private static final long WINDOW_SECONDS = 3600;
    
    public boolean isAllowed(String clientId, String endpoint) {
        String key = "rate_limit:" + clientId + ":" + endpoint;
        
        Integer count = redisTemplate.opsForValue().get(key);
        
        if (count == null) {
            redisTemplate.opsForValue().set(key, 1, Duration.ofSeconds(WINDOW_SECONDS));
            return true;
        }
        
        if (count >= MAX_REQUESTS) {
            return false;
        }
        
        redisTemplate.opsForValue().increment(key);
        return true;
    }
    
    public int getRemainingRequests(String clientId, String endpoint) {
        String key = "rate_limit:" + clientId + ":" + endpoint;
        Integer count = redisTemplate.opsForValue().get(key);
        return Math.max(0, MAX_REQUESTS - (count != null ? count : 0));
    }
}
```

---

## Hard Level Questions

#### Q11: Design a self-healing data pipeline that automatically detects and recovers from schema changes.

**Answer:**

```java
@Service
public class SelfHealingPipeline {
    
    @Autowired
    private SchemaRegistry schemaRegistry;
    
    @Autowired
    private DataValidator validator;
    
    @Autowired
    private RecoveryOrchestrator recovery;
    
    @Autowired
    private AlertService alertService;
    
    @KafkaListener(topics = "raw-data")
    public void processWithSchemaValidation(ConsumerRecord<String, String> record) {
        try {
            String data = record.value();
            String sourceSystem = record.headers().lastHeader("source-system").value().toString();
            
            // Step 1: Validate against current schema
            SchemaValidationResult validation = validateSchema(data, sourceSystem);
            
            if (validation.isValid()) {
                processData(data);
            } else {
                handleSchemaIssue(validation, data, sourceSystem);
            }
            
        } catch (SchemaEvolutionException e) {
            recovery.handleSchemaEvolution(e);
        }
    }
    
    private SchemaValidationResult validateSchema(String data, String sourceSystem) {
        // Fetch latest schema
        Schema currentSchema = schemaRegistry.getLatestSchema(sourceSystem);
        
        try {
            // Parse and validate
            GenericRecord record = parseData(data, currentSchema);
            return SchemaValidationResult.valid();
        } catch (AvroTypeException e) {
            // Schema mismatch detected
            return SchemaValidationResult.invalid("Schema mismatch: " + e.getMessage());
        }
    }
    
    private void handleSchemaIssue(SchemaValidationResult validation, 
                                   String data, String sourceSystem) {
        logger.warn("Schema validation failed: {}", validation.getError());
        
        // Try to auto-recover
        if (canAutoRecover(validation)) {
            String recoveredData = attemptAutoRecovery(data, sourceSystem);
            if (recoveredData != null) {
                processData(recoveredData);
                alertService.sendInfo("Auto-recovered from schema mismatch");
                return;
            }
        }
        
        // If auto-recovery fails, route to reconciliation service
        sendToReconciliation(data, sourceSystem, validation);
        alertService.sendAlert("Manual schema fix required for: " + sourceSystem);
    }
    
    private boolean canAutoRecover(SchemaValidationResult validation) {
        // Recoverable: missing optional fields, extra fields, type coercion
        return validation.getError().contains("missing optional") ||
               validation.getError().contains("extra field") ||
               validation.getError().contains("type mismatch");
    }
    
    private String attemptAutoRecovery(String data, String sourceSystem) {
        try {
            Schema currentSchema = schemaRegistry.getLatestSchema(sourceSystem);
            ObjectMapper mapper = new ObjectMapper();
            JsonNode node = mapper.readTree(data);
            
            // Remove extra fields not in schema
            JsonNode cleaned = removeExtraFields(node, currentSchema);
            
            // Add missing optional fields with defaults
            JsonNode withDefaults = addMissingDefaults(cleaned, currentSchema);
            
            return mapper.writeValueAsString(withDefaults);
            
        } catch (Exception e) {
            logger.error("Auto-recovery failed", e);
            return null;
        }
    }
}

@Service
public class RecoveryOrchestrator {
    
    @Autowired
    private SchemaRegistry schemaRegistry;
    
    @Autowired
    private DataLakeService dataLake;
    
    public void handleSchemaEvolution(SchemaEvolutionException e) {
        logger.info("Schema evolution detected: {} -> {}",
            e.getOldSchema(), e.getNewSchema());
        
        // Step 1: Detect the type of change
        SchemaChangeType changeType = detectChangeType(e.getOldSchema(), e.getNewSchema());
        
        // Step 2: Route to appropriate handler
        switch (changeType) {
            case COLUMN_ADDED:
                handleColumnAddition(e);
                break;
            case COLUMN_REMOVED:
                handleColumnRemoval(e);
                break;
            case TYPE_CHANGED:
                handleTypeChange(e);
                break;
            case RENAMED:
                handleColumnRename(e);
                break;
        }
    }
    
    private void handleColumnAddition(SchemaEvolutionException e) {
        // If optional (nullable), safe - just provide default
        // If required, need to backfill historical data
        
        Schema old = e.getOldSchema();
        Schema new_schema = e.getNewSchema();
        Field newField = findNewField(old, new_schema);
        
        if (newField.isOptional()) {
            logger.info("New optional field: {}, using default", newField.getName());
        } else {
            logger.warn("New required field: {}, need backfill", newField.getName());
            triggerBackfillJob(newField);
        }
    }
    
    private void handleTypeChange(SchemaEvolutionException e) {
        // Check if conversion is safe
        Type oldType = e.getOldSchema().getFieldType(e.getFieldName());
        Type newType = e.getNewSchema().getFieldType(e.getFieldName());
        
        if (canConvert(oldType, newType)) {
            logger.info("Type change is compatible: {} -> {}", oldType, newType);
        } else {
            logger.error("Incompatible type change, manual intervention needed");
            triggerManualReview(e);
        }
    }
}
```

---

#### Q12: How would you implement a real-time change data capture system using Java?

**Answer:**

```java
@Service
public class RealtimeCDCSystem {
    
    @Autowired
    private DebeziumConnectorService debezium;
    
    @Autowired
    private KafkaTemplate<String, ChangeEvent> kafkaTemplate;
    
    @Autowired
    private CDCMetricsService metrics;
    
    // Configure Debezium connector
    public void setupDebezium() {
        Map<String, String> config = new HashMap<>();
        config.put("name", "postgres-cdc");
        config.put("connector.class", "io.debezium.connector.postgresql.PostgresConnector");
        config.put("database.hostname", "localhost");
        config.put("database.port", "5432");
        config.put("database.user", "replication_user");
        config.put("database.password", "password");
        config.put("database.dbname", "mydb");
        config.put("database.server.name", "production");
        config.put("table.include.list", "public.customers,public.orders");
        config.put("plugin.name", "pgoutput");
        config.put("publication.name", "dbz_publication");
        config.put("slot.name", "dbz_slot");
        
        debezium.createConnector(config);
    }
    
    @KafkaListener(topics = "cdc\\..*", groupId = "realtime-cdc")
    public void captureChanges(ConsumerRecord<String, ChangeEvent> record) throws Exception {
        ChangeEvent event = record.value();
        
        long startTime = System.currentTimeMillis();
        
        try {
            logger.info("Captured change: op={}, table={}, id={}",
                event.getOp(), event.getTable(), event.getKey());
            
            // Enrich change event with metadata
            enrichChangeEvent(event);
            
            // Process based on operation
            switch (event.getOp()) {
                case "c": // Create
                    handleInsert(event);
                    break;
                case "u": // Update
                    handleUpdate(event);
                    break;
                case "d": // Delete
                    handleDelete(event);
                    break;
                case "r": // Read
                    handleRead(event);
                    break;
            }
            
            long latency = System.currentTimeMillis() - startTime;
            metrics.recordLatency(latency);
            
        } catch (Exception e) {
            logger.error("Failed to process CDC event", e);
            sendToErrorTopic(event, e);
        }
    }
    
    private void enrichChangeEvent(ChangeEvent event) {
        event.setProcessedAt(Instant.now());
        event.setLatencyMs(calculateLatency(event));
    }
    
    private void handleInsert(ChangeEvent event) {
        // After values represent the new row
        Map<String, Object> newData = event.getAfter();
        
        // Update all destinations
        updateSearchIndex(newData);
        updateCache(newData);
        updateAnalyticsDB(newData);
    }
    
    private void handleUpdate(ChangeEvent event) {
        Map<String, Object> oldData = event.getBefore();
        Map<String, Object> newData = event.getAfter();
        
        // Find what changed
        Map<String, Object> changes = findChanges(oldData, newData);
        
        if (!changes.isEmpty()) {
            invalidateCache(oldData.get("id"));
            updateSearchIndex(newData);
            trackAudit(oldData, newData, changes);
        }
    }
    
    private void handleDelete(ChangeEvent event) {
        Map<String, Object> deletedData = event.getBefore();
        
        removeFromCache(deletedData.get("id"));
        removeFromSearchIndex(deletedData.get("id"));
        trackDeletion(deletedData);
    }
}

@Service
public class CDCMetricsService {
    
    @Autowired
    private MeterRegistry meterRegistry;
    
    Timer cdcLatency = Timer.builder("cdc.processing.latency")
        .description("Time to process CDC events")
        .publishPercentiles(0.5, 0.95, 0.99)
        .register(meterRegistry);
    
    Counter cdcEventsProcessed = Counter.builder("cdc.events.processed")
        .description("Total CDC events processed")
        .register(meterRegistry);
    
    public void recordLatency(long latencyMs) {
        cdcLatency.record(latencyMs, TimeUnit.MILLISECONDS);
    }
    
    public void recordEvent() {
        cdcEventsProcessed.increment();
    }
}
```

---

#### Q13: Design a system that optimizes query performance across multiple heterogeneous data sources.

**Answer:**

```java
@Service
public class FederatedQueryEngine {
    
    @Autowired
    private DataSourceRegistry registry;
    
    @Autowired
    private QueryPlanner planner;
    
    @Autowired
    private QueryExecutor executor;
    
    @Autowired
    private ResultMerger merger;
    
    /**
     * Execute a query that may span multiple data sources
     */
    public ResultSet executeQuery(String query) {
        // Step 1: Parse query
        ParsedQuery parsed = QueryParser.parse(query);
        
        // Step 2: Identify required data sources
        Set<String> sources = identifyDataSources(parsed);
        logger.info("Query requires sources: {}", sources);
        
        // Step 3: Create execution plan
        ExecutionPlan plan = planner.createOptimalPlan(parsed, sources);
        logExecutionPlan(plan);
        
        // Step 4: Execute in parallel where possible
        Map<String, PartialResult> results = executor.executeParallel(plan);
        
        // Step 5: Merge and sort results
        return merger.mergeResults(results, parsed.getOrderBy(), parsed.getLimit());
    }
    
    private Set<String> identifyDataSources(ParsedQuery query) {
        Set<String> sources = new HashSet<>();
        
        for (String table : query.getTables()) {
            // Lookup table location in metadata
            DataSourceInfo info = registry.getDataSourceForTable(table);
            sources.add(info.getName());
        }
        
        return sources;
    }
}

@Service
public class QueryPlanner {
    
    /**
     * Create optimal execution plan considering:
     * 1. Push predicates down to source
     * 2. Minimize data transfer
     * 3. Execute fast sources first
     * 4. Parallelize independent queries
     */
    public ExecutionPlan createOptimalPlan(ParsedQuery query, Set<String> sources) {
        ExecutionPlan plan = new ExecutionPlan();
        
        for (String source : sources) {
            DataSourceInfo info = getDataSourceInfo(source);
            
            // Create sub-query optimized for this source
            SubQuery subQuery = createOptimizedSubQuery(query, source, info);
            
            // Calculate estimated cost
            Cost cost = estimateCost(subQuery, info);
            
            ExecutionNode node = ExecutionNode.builder()
                .source(source)
                .subQuery(subQuery)
                .estimatedCost(cost)
                .priority(calculatePriority(cost))
                .build();
            
            plan.addNode(node);
        }
        
        // Sort by priority (parallel fast queries, wait for slow)
        plan.sortByPriority();
        
        return plan;
    }
    
    private SubQuery createOptimizedSubQuery(ParsedQuery query, String source, DataSourceInfo info) {
        SubQuery sub = new SubQuery();
        
        // List SELECT columns (minimize transfer)
        sub.setColumns(query.getColumns());
        
        // Add source-specific predicates
        List<Predicate> pushdownPredicates = query.getPredicates().stream()
            .filter(p -> info.supportsFunction(p.getFunctionType()))
            .collect(Collectors.toList());
        
        sub.setWhere(pushdownPredicates);
        
        // Hint for optimizer
        if (info.isDataWarehouse()) {
            sub.addHint("USE COLUMNAR_SCAN");
        } else if (info.isElasticsearch()) {
            sub.addHint("USE_LUCENE_QUERY");
        }
        
        return sub;
    }
}

@Service
public class QueryExecutor {
    
    public Map<String, PartialResult> executeParallel(ExecutionPlan plan) {
        Map<String, PartialResult> results = new ConcurrentHashMap<>();
        List<CompletableFuture<PartialResult>> futures = new ArrayList<>();
        
        for (ExecutionNode node : plan.getNodes()) {
            CompletableFuture<PartialResult> future = CompletableFuture.supplyAsync(
                () -> executeNode(node)
            );
            
            futures.add(future);
        }
        
        // Wait for all to complete
        CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join();
        
        // Collect results
        futures.forEach(f -> {
            PartialResult result = f.join();
            results.put(result.getSource(), result);
        });
        
        return results;
    }
    
    private PartialResult executeNode(ExecutionNode node) {
        DataSourceInfo info = registry.getDataSourceForTable(node.getSource());
        
        try {
            switch (info.getType()) {
                case "PostgreSQL":
                    return executePostgresQuery(node, info);
                case "Elasticsearch":
                    return executeElasticsearchQuery(node, info);
                case "BigQuery":
                    return executeBigQueryQuery(node, info);
                case "S3":
                    return executeS3Query(node, info);
                default:
                    throw new UnsupportedQuerySourceException(info.getType());
            }
        } catch (QueryTimeoutException e) {
            logger.warn("Query timeout for source: {}", node.getSource());
            return PartialResult.timeout(node.getSource());
        }
    }
    
    private PartialResult executePostgresQuery(ExecutionNode node, DataSourceInfo info) {
        try (Connection conn = info.getDataSource().getConnection();
             Statement stmt = conn.createStatement()) {
            
            stmt.setQueryTimeout(30); // 30 seconds
            
            String sql = node.getSubQuery().toSQL();
            ResultSet rs = stmt.executeQuery(sql);
            
            List<Map<String, Object>> data = resultSetToList(rs);
            return PartialResult.success(node.getSource(), data);
            
        } catch (SQLException e) {
            return PartialResult.error(node.getSource(), e);
        }
    }
    
    private PartialResult executeElasticsearchQuery(ExecutionNode node, DataSourceInfo info) {
        RestHighLevelClient client = info.getElasticsearchClient();
        
        SearchRequest request = new SearchRequest(node.getSubQuery().getIndex());
        SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
        
        sourceBuilder.query(QueryBuilders.matchAllQuery());
        sourceBuilder.size(10000);
        
        request.source(sourceBuilder);
        
        try {
            SearchResponse response = client.search(request, RequestOptions.DEFAULT);
            
            List<Map<String, Object>> data = new ArrayList<>();
            for (SearchHit hit : response.getHits()) {
                data.add(hit.getSourceAsMap());
            }
            
            return PartialResult.success(node.getSource(), data);
            
        } catch (IOException e) {
            return PartialResult.error(node.getSource(), e);
        }
    }
}

@Service
public class ResultMerger {
    
    public ResultSet mergeResults(Map<String, PartialResult> results, 
                                 List<OrderBy> orderBy, int limit) {
        List<Map<String, Object>> merged = new ArrayList<>();
        
        // Merge all result sets
        for (PartialResult result : results.values()) {
            if (result.isSuccess()) {
                merged.addAll(result.getData());
            } else if (result.isTimeout()) {
                logger.warn("Partial timeout: {}", result.getSource());
                // Use best effort: continue with other results
            }
        }
        
        // Apply post-query sorting
        for (OrderBy order : orderBy) {
            merged.sort((a, b) -> {
                Object valA = a.get(order.getColumn());
                Object valB = b.get(order.getColumn());
                
                int cmp = compareValues(valA, valB);
                return order.isDescending() ? -cmp : cmp;
            });
        }
        
        // Apply limit
        if (limit > 0 && merged.size() > limit) {
            merged = merged.subList(0, limit);
        }
        
        return new ResultSet(merged);
    }
}
```

---

#### Q14: Explain how you'd build a data lakehouse architecture and the role of Java applications in it.

**Answer:**

```
Data Lakehouse Architecture:

Layer 1: Ingestion (Java Apps)
        ↓
Layer 2: Bronze (Raw Data)
        ↓
Layer 3: Silver (Cleaned Data)   <- Metadata/Governance
        ↓
Layer 4: Gold (Ready for Use)     <- Business Tables
        ↓
Layer 5: Consumption (Analytics, ML, BI)

Java Role: Orchestrate entire flow
```

```java
@Service
public class DataLakehouseOrchestrator {
    
    @Autowired
    private BronzeLayerService bronzeService;
    
    @Autowired
    private SilverLayerService silverService;
    
    @Autowired
    private GoldLayerService goldService;
    
    @Autowired
    private MetadataService metadataService;
    
    @Scheduled(cron = "0 0 * * * ?")
    public void orchestrateDataRefresh() {
        logger.info("Starting Lakehouse refresh cycle");
        
        try {
            // Stage 1: Ingest to Bronze
            Map<String, Long> bronzeMetrics = bronzeService.ingestRawData();
            logger.info("Bronze ingestion complete: {}", bronzeMetrics);
            
            // Stage 2: Transform to Silver
            Map<String, Long> silverMetrics = silverService.transformData();
            logger.info("Silver transformation complete: {}", silverMetrics);
            
            // Stage 3: Aggregate to Gold
            Map<String, Long> goldMetrics = goldService.aggregateData();
            logger.info("Gold aggregation complete: {}", goldMetrics);
            
            // Update metadata
            metadataService.recordCycle(bronzeMetrics, silverMetrics, goldMetrics);
            
            // Quality checks
            DataQualityReport report = metadataService.validateQuality();
            if (!report.isAcceptable()) {
                alertService.sendAlert("Data Quality Issues", report.getIssues());
            }
            
            logger.info("Lakehouse refresh completed successfully");
            
        } catch (Exception e) {
            logger.error("Lakehouse refresh failed", e);
            throw new LakehouseException("Refresh failed", e);
        }
    }
}

@Service
public class BronzeLayerService {
    
    @Autowired
    private DataSourceRegistry registry;
    
    @Autowired
    private AmazonS3 s3Client;
    
    public Map<String, Long> ingestRawData() {
        Map<String, Long> metrics = new HashMap<>();
        LocalDate today = LocalDate.now();
        
        // Ingest from all registered sources
        for (DataSourceInfo source : registry.getAllSources()) {
            try {
                long count = ingestFromSource(source, today);
                metrics.put(source.getName(), count);
                
            } catch (Exception e) {
                logger.error("Failed to ingest from: {}", source.getName(), e);
                metrics.put(source.getName(), -1L);
            }
        }
        
        return metrics;
    }
    
    private long ingestFromSource(DataSourceInfo source, LocalDate date) throws Exception {
        String path = String.format("s3://datalake/bronze/%s/%s/",
            source.getName(), date);
        
        switch (source.getType()) {
            case "database":
                return ingestFromDatabase(source, path);
            case "api":
                return ingestFromAPI(source, path);
            case "kafka":
                return ingestFromKafka(source, path);
            default:
                throw new UnsupportedSourceException(source.getType());
        }
    }
    
    private long ingestFromDatabase(DataSourceInfo source, String path) throws Exception {
        List<Map<String, Object>> data = fetchFromDatabase(source);
        
        // Write as Parquet (compressed, columnar)
        String key = path + "data_" + System.currentTimeMillis() + ".parquet";
        writeParquetToS3(data, key);
        
        return data.size();
    }
}

@Service
public class SilverLayerService {
    
    @Autowired
    private AmazonS3 s3Client;
    
    @Autowired
    private DataValidator validator;
    
    public Map<String, Long> transformData() {
        Map<String, Long> metrics = new HashMap<>();
        LocalDate today = LocalDate.now();
        
        // Process each table
        List<String> tables = Arrays.asList("customers", "orders", "products");
        
        for (String table : tables) {
            try {
                String bronzePath = String.format("s3://datalake/bronze/*/%s/", table);
                String silverPath = String.format("s3://datalake/silver/%s/%s/", table, today);
                
                long count = processTable(bronzePath, silverPath, table);
                metrics.put(table, count);
                
            } catch (Exception e) {
                logger.error("Failed to transform table: {}", table, e);
                metrics.put(table, -1L);
            }
        }
        
        return metrics;
    }
    
    private long processTable(String bronzePath, String silverPath, String table) throws Exception {
        // Read from Bronze using Spark
        Dataset<Row> df = readParquetFromS3(bronzePath);
        
        // Apply transformations
        df = df
            .filter("is_valid = true") // Remove invalid records
            .dropDuplicates(new String[]{"id"}) // Remove duplicates
            .na().fill(createDefaultValues(table)); // Fill NULLs
        
        // Apply business logic
        df = enrichData(df, table);
        
        // Write to Silver (partitioned for performance)
        df.repartition(100)
            .write()
            .mode("overwrite")
            .parquet(silverPath);
        
        return df.count();
    }
}

@Service
public class GoldLayerService {
    
    public Map<String, Long> aggregateData() {
        Map<String, Long> metrics = new HashMap<>();
        LocalDate today = LocalDate.now();
        
        // Create business-ready tables
        metrics.put("customer_360", buildCustomer360());
        metrics.put("orders_summary", buildOrdersSummary());
        metrics.put("product_analytics", buildProductAnalytics());
        
        return metrics;
    }
    
    private long buildCustomer360() {
        // Join silver layer tables into comprehensive customer view
        Dataset<Row> customers = readParquetFromS3("s3://datalake/silver/customers/");
        Dataset<Row> orders = readParquetFromS3("s3://datalake/silver/orders/");
        
        Dataset<Row> customer360 = customers
            .join(orders, customers.col("id").equalTo(orders.col("customer_id")), "left")
            .groupBy("id")
            .agg(
                first("name").as("customer_name"),
                count("order_id").as("total_orders"),
                sum("amount").as("lifetime_value"),
                max("order_date").as("last_order_date")
            );
        
        // Write to Gold layer
        customer360
            .write()
            .mode("overwrite")
            .parquet("s3://datalake/gold/customer_360/");
        
        return customer360.count();
    }
}

@Service
public class MetadataService {
    
    @Autowired
    private MetadataRepository repository;
    
    public void recordCycle(Map<String, Long> bronzeMetrics, 
                           Map<String, Long> silverMetrics,
                           Map<String, Long> goldMetrics) {
        
        LakehouseCycle cycle = LakehouseCycle.builder()
            .cycleDate(LocalDate.now())
            .timestamp(LocalDateTime.now())
            .bronzeMetrics(bronzeMetrics)
            .silverMetrics(silverMetrics)
            .goldMetrics(goldMetrics)
            .status("SUCCESS")
            .build();
        
        repository.save(cycle);
        
        // Update data catalog with table locations
        updateDataCatalog(bronzeMetrics, silverMetrics, goldMetrics);
    }
    
    public DataQualityReport validateQuality() {
        DataQualityReport report = new DataQualityReport();
        
        // Check completeness
        checkCompleteness(report);
        
        // Check accuracy
        checkAccuracy(report);
        
        // Check consistency
        checkConsistency(report);
        
        return report;
    }
}
```

---

#### Q15: Design a multi-tenant data isolation strategy considering performance and compliance requirements.

**Answer:**

```java
/**
 * Multi-tenant isolation requires:
 * 1. Logical isolation: Tenant ID in every query
 * 2. Physical isolation: Separate schemas/databases if needed
 * 3. Row-level security: Filter by tenant
 * 4. Encryption: Per-tenant encryption keys
 * 5. Compliance: GDPR, data residency
 */

@Configuration
public class MultiTenantConfig {
    
    @Bean
    public TenantInterceptor tenantInterceptor() {
        return new TenantInterceptor();
    }
    
    @Bean
    public TenantAwarePersistenceUnitManager persistenceUnitManager() {
        return new TenantAwarePersistenceUnitManager();
    }
}

@Component
public class TenantContext {
    
    private static final ThreadLocal<String> currentTenant = new ThreadLocal<>();
    
    public static void setCurrentTenant(String tenantId) {
        currentTenant.set(tenantId);
    }
    
    public static String getCurrentTenant() {
        String tenant = currentTenant.get();
        if (tenant == null) {
            throw new TenantNotFoundException("No tenant context set");
        }
        return tenant;
    }
    
    public static void clear() {
        currentTenant.remove();
    }
}

@Component
public class TenantInterceptor implements WebRequestInterceptor {
    
    @Override
    public void preHandle(ServletRequest request) throws Exception {
        String tenantId = extractTenantId(request);
        if (tenantId == null) {
            throw new InvalidTenantException("Missing X-Tenant-ID header");
        }
        
        TenantContext.setCurrentTenant(tenantId);
    }
    
    private String extractTenantId(ServletRequest request) {
        // Option 1: From header
        String header = request.getHeader("X-Tenant-ID");
        if (header != null) return header;
        
        // Option 2: From JWT
        String jwt = extractJWT(request);
        return extractTenantFromJWT(jwt);
    }
    
    @Override
    public void postHandle(ServletRequest request, ServletResponse response) {
        TenantContext.clear();
    }
}

@Service
public class TenantAwareOrderService {
    
    @Autowired
    private OrderRepository repository;
    
    @Autowired
    private TenantEncryptionService encryptionService;
    
    @Transactional
    public Order createOrder(OrderRequest request) {
        String tenantId = TenantContext.getCurrentTenant();
        
        // All orders created with tenant context
        Order order = new Order();
        order.setTenantId(tenantId);
        order.setCustomerId(request.getCustomerId());
        order.setAmount(request.getAmount());
        
        // Encrypt sensitive fields
        order.setSensitiveData(encrypt(request.getSensitiveData(), tenantId));
        
        return repository.save(order);
    }
    
    @Transactional(readOnly = true)
    public List<Order> getOrders() {
        String tenantId = TenantContext.getCurrentTenant();
        
        // Automatically filtered by tenant
        return repository.findByTenantId(tenantId);
    }
    
    private String encrypt(String data, String tenantId) {
        return encryptionService.encrypt(data, tenantId);
    }
}

@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {
    
    // Tenant-aware queries
    @Query("SELECT o FROM Order o WHERE o.tenantId = :tenantId")
    List<Order> findByTenantId(@Param("tenantId") String tenantId);
    
    @Query("SELECT o FROM Order o WHERE o.tenantId = :tenantId AND o.id = :id")
    Optional<Order> findByIdAndTenant(@Param("id") Long id, @Param("tenantId") String tenantId);
}

@Service
public class TenantEncryptionService {
    
    @Autowired
    private KeyManagementService kmsService;
    
    public String encrypt(String data, String tenantId) {
        // Get tenant-specific encryption key
        String key = kmsService.getKeyForTenant(tenantId);
        
        try {
            Cipher cipher = Cipher.getInstance("AES");
            SecretKey secretKey = new SecretKeySpec(key.getBytes(), 0, 16, "AES");
            cipher.init(Cipher.ENCRYPT_MODE, secretKey);
            
            byte[] encrypted = cipher.doFinal(data.getBytes());
            return Base64.getEncoder().encodeToString(encrypted);
            
        } catch (Exception e) {
            throw new EncryptionException("Encryption failed", e);
        }
    }
    
    public String decrypt(String encryptedData, String tenantId) {
        String key = kmsService.getKeyForTenant(tenantId);
        
        try {
            Cipher cipher = Cipher.getInstance("AES");
            SecretKey secretKey = new SecretKeySpec(key.getBytes(), 0, 16, "AES");
            cipher.init(Cipher.DECRYPT_MODE, secretKey);
            
            byte[] decodedData = Base64.getDecoder().decode(encryptedData);
            byte[] decrypted = cipher.doFinal(decodedData);
            return new String(decrypted);
            
        } catch (Exception e) {
            throw new DecryptionException("Decryption failed", e);
        }
    }
}

@Service
public class TenantDataResidencyService {
    
    /**
     * Ensure data residency compliance per tenant
     * Example: EU tenants -> EU data centers only
     */
    public String getDataSourceForTenant(String tenantId) {
        TenantInfo tenant = getTenantInfo(tenantId);
        
        switch (tenant.getDataResidency()) {
            case "EU":
                return "eu-postgres-primary"; // EU data center
            case "US":
                return "us-postgres-primary"; // US data center
            case "APAC":
                return "apac-postgres-primary"; // Asia-Pacific
            default:
                return "default-postgres";
        }
    }
    
    public void validateDataResidency(String tenantId, String region) {
        TenantInfo tenant = getTenantInfo(tenantId);
        
        if (!Arrays.asList(tenant.getAllowedRegions()).contains(region)) {
            throw new ComplianceViolationException(
                String.format("Tenant %s cannot access region %s", tenantId, region)
            );
        }
    }
}

@Service
public class TenantDataGovernanceService {
    
    @Autowired
    private AuditLogRepository auditLog;
    
    @Transactional
    public void logDataAccess(String tenantId, String dataId, String operation, String userId) {
        AuditLog log = AuditLog.builder()
            .tenantId(tenantId)
            .dataId(dataId)
            .operation(operation)
            .userId(userId)
            .timestamp(LocalDateTime.now())
            .build();
        
        auditLog.save(log);
    }
    
    public void ensureGDPRCompliance(String tenantId) {
        // Right to be forgotten: Delete all data for tenant
        // Right to data portability: Export all data
        // Data minimization: Only store necessary fields
    }
}

// Schema-per-tenant approach (for stronger isolation)
@Configuration
public class SchemaDrivenTenantResolver implements TenantResolver {
    
    @Override
    public String resolveTenantId() {
        String tenantId = TenantContext.getCurrentTenant();
        
        // Map tenant to schema
        // tenant1 -> schema_tenant1
        // tenant2 -> schema_tenant2
        
        return "schema_" + tenantId;
    }
}

//Database-per-tenant approach (maximum isolation)
@Service
public class DatabasePerTenantService {
    
    @Autowired
    private TenantRegistry tenantRegistry;
    
    public DataSource getDataSourceForTenant(String tenantId) {
        TenantDatabaseMapping mapping = tenantRegistry.getMapping(tenantId);
        
        // Create or retrieve datasource for specific database
        return createDataSource(
            mapping.getHost(),
            mapping.getPort(),
            mapping.getDatabaseName(),
            mapping.getUsername(),
            mapping.getPassword()
        );
    }
}
```

---

## Recommended Reading/Skills to Brush Up

### Database Fundamentals

**Critical Topics:**
1. SQL Optimization
   - EXPLAIN PLAN analysis
   - Index selection
   - Query rewriting
   - Statistics gathering
   
   **Practice:**
   - Run 10 slow queries, optimize each
   - Understand execution plans
   - Know B-tree vs Bitmap index differences

2. ACID Properties
   - Atomicity: All-or-nothing transactions
   - Consistency: Data integrity rules
   - Isolation: Concurrent execution safety
   - Durability: Persistent storage
   
   **Practice:**
   - Write code with transaction boundaries
   - Handle transaction failures gracefully
   - Test concurrent updates

3. Indexing Strategies
   - Composite indexes
   - Index selectivity
   - Covering indexes
   - Index maintenance cost
   
   **Study:**
   - How indexes impact INSERT/UPDATE performance
   - When NOT to add indexes
   - Index fragmentation issues

### Data Concepts

**Essential Knowledge:**
1. ETL vs ELT
   - Old way (ETL): Extract, Transform in middleware, Load
   - Modern way (ELT): Extract, Load, Transform in warehouse
   - When to use each

2. Data Modeling
   - Kimball (dimensional): Analytics focus
   - Inmon (3NF): Traditional normalization
   - Data Vault: Scalable enterprise
   
   **Hands-on:**
   - Design a 50+ table warehouse schema
   - Understand SCD implementations
   - Know dimension key generation strategies

3. Big Data Fundamentals
   - MapReduce: Split, map, shuffle, reduce, combine
   - Spark: In-memory distributed processing
   - Partitioning: How data is distributed
   - Skew handling: Hotspot mitigation

### Java-Specific Skills

**Must Know:**
1. JDBC Deep Dive
   ```java
   - Connection pooling configuration
   - PreparedStatement benefits
   - ResultSet pagination
   - Batch operations
   - Connection lifecycle
   ```

2. Hibernate/JPA Mastery
   ```java
   - Entity state transitions
   - Session/PersistenceContext management
   - N+1 query problem solutions
   - Query caching strategies
   - Lazy/eager loading implications
   ```

3. Spring Data Excellence
   ```java
   - Repository pattern understanding
   - Query derivation
   - Custom queries (@Query)
   - Projections and DTO mapping
   - Pagination and sorting
   ```

4. Stream API for Data Processing
   ```java
   - filter(), map(), flatMap()
   - Lazy evaluation
   - Parallel streams (use carefully!)
   - Custom collectors
   - Terminal operations
   ```

5. Concurrent Programming
   ```java
   - ExecutorService
   - CompletableFuture
   - Thread pools
   - Synchronization primitives
   - Non-blocking I/O
   ```

### AWS Data Services

**Hands-on Practice:**
1. **RDS Aurora**
   - Create instances
   - Configure read replicas
   - Monitor performance
   - Implement connection pooling
   - Handle failover

2. **DynamoDB**
   - Design access patterns
   - Understand GSI/LSI
   - Handle throttling
   - Cost optimization
   - TTL implementation

3. **S3 and Data Lakes**
   - Partitioning strategy: `s3://bucket/year=2024/month=03/day=06/`
   - Lifecycle policies
   - Encryption options
   - Access logging
   - Transition management

4. **Redshift**
   - Understand columnar storage
   - Distribution and sort keys
   - Query performance tuning
   - Spectrum for S3 querying
   - Workload management

5. **IAM and Security**
   - Principle of least privilege
   - Cross-account access
   - Temporary credentials
   - KMS encryption
   - VPC endpoints

### Tools & Technologies

**To Get Proficient With:**
1. **Monitoring & Observability**
   - Prometheus: Metrics
   - Grafana: Dashboards
   - ELK Stack: Logs
   - Datadog: APM

2. **Message Queues**
   - Kafka: Stream processing
   - RabbitMQ: Task queues
   - AWS SQS/SNS: AWS native

3. **Containerization**
   - Docker: Container basics
   - Docker Compose: Multi-container apps
   - Kubernetes: Orchestration (optional)

---

## How to Position Yourself

### 1. Show Data Awareness

```
BAD Interview Approach:
"I wrote a Java endpoint that queries the database"

GOOD Approach:
"I designed an endpoint that fetches data efficiently by:
- Using pagination (cursor-based for 1M+ records)
- Lazy-loading relationships to avoid N+1 queries
- Caching frequent queries with 5-min TTL
- Monitoring query latency (p95: 100ms target)
- Testing with realistic data volumes"
```

### 2. Ask Smart Questions

During the interview, ask questions that show data understanding:

```
"If we scale to 100M records:
- Should we consider sharding? What key would we use?
- How would we handle replication lag in read replicas?
- Would batch vs streaming ingestion be better?
- What metrics matter most: latency, consistency, or cost?
- How do you currently monitor data quality?"
```

### 3. Share Concrete Examples

Prepare 3-5 real examples from your work:

```
Example 1: Query Optimization
"Problem: Customer report took 45 minutes
Root cause: N+1 queries (1M customers, each with 10 addresses)
Solution: Single join query + caching -> 2 seconds
Learning: Always think about query patterns at scale"

Example 2: Data Quality
"Problem: Duplicate orders in warehouse
Root cause: ETL non-idempotent, ran twice accidentally
Solution: Added transaction IDs, made pipeline idempotent
Impact: Prevented $50K in duplicate charges"

Example 3: Performance
"Problem: Data pipeline failing after 2 hours
Root cause: Loading 2GB result set into memory
Solution: Stream processing, batch inserts of 10K records
Result: Completed in 20 minutes, 10% memory usage"
```

### 4. Demonstrate End-to-End Understanding

Show how you think about complete data flow:

```
User Request
  ↓
(Discuss) Java REST API with param validation
  ↓
(Discuss) Database query optimization + caching
  ↓
(Discuss) Data consistency + error handling
  ↓
(Discuss) Monitoring + alerting
  ↓
(Discuss) Analytics pipeline updates
  ↓
(Discuss) BI dashboard refresh
```

Show the interviewer you understand their world:
- "How does this feed your data warehouse?"
- "Would this need to be replicated across regions?"
- "What are the data quality implications?"
- "How would this scale to 10B records?"

### 5. Performance Mindset

Always analyze trade-offs:

```
Decision: In-memory vs on-disk processing
Analysis:
- 100M records: Would need 40GB RAM -> not practical in-memory
- Option 1: Stream + batch (5K per batch) -> 20K batches
- Option 2: Database native filtering -> single optimized query
- Option 3: Spark distributed processing
- Recommendation: Database native filtering
  - Lowest latency (uses indexes)
  - Lowest cost (no extra infrastructure)
  - Highest reliability (time-tested technology)
```

### 6. Code Quality Shows Character

Write Java code that shows thinking:

```java
// Bad: Naive approach
for (Order order : allOrders) {
    customer = fetchCustomer(order.getCustomerId()); // N queries!
}

// Good: Data-aware approach
List<Order> orders = orderRepository.findWithCustomer(spec, pageable);
List<Long> customerIds = orders.stream()
    .map(Order::getCustomerId)
    .distinct()
    .collect(toList());

Map<Long, Customer> customers = customerRepository.findAllById(customerIds)
    .stream()
    .collect(toMap(Customer::getId, identity()));
```

### 7. Discuss Trade-offs with Confidence

The interviewer will respect thoughtful analysis:

```
"For this use case, here are the options:

1. Real-time (Kafka + Spark Streaming)
   - Pro: Millisecond latency, live dashboards
   - Con: Complex, 24/7 operations, $150K/month
   - When: Stock prices, fraud detection

2. Batch (Nightly job)
   - Pro: Simple, reliable, $10K/month
   - Con: 24-hour latency, not suitable for real-time
   - When: Daily reporting, analytics

3. Micro-batching (5-min windows)
   - Pro: Good balance, $40K/month
   - Con: Not true real-time
   - When: This use case!
   
I'd recommend option 3 because:
- 5-min latency meets business SLA
- Leverages batch technology (Spark) for stability
- Cost-effective with Fargate auto-scaling
"
```

### 8. Talk about Failures and Learning

This impresses data engineers most:

```
"In my previous role, our ETL pipeline failed silently:
- Problem: No validation layer, bad data loaded to warehouse
- Impact: Sales reports wrong for 3 months
- Root Cause: Assumed upstream data quality
- Solution: Built validation framework
  - Check completeness (NULL count)
  - Check cardinality (distinct values)
  - Check business rules (amount >= 0)
  - Alert if quality < 95%
- Result: Zero data quality incidents next 2 years
- Learning: Data quality is feature, not afterthought"
```

### 9. Demonstrate System-Thinking

Show you understand the bigger picture:

```
"When building this microservice, I considered:
1. Database impact: Will this create hotspots?
2. Data pipeline impact: How does this feed analytics?
3. Compliance: GDPR, data retention, encryption
4. Scalability: What breaks at 1M concurrent users?
5. Monitoring: What metrics predict failure?
6. Recovery: Can we replay if something breaks?
7. Cost: How much will this cost at 10x scale?"
```

### 10. Show Respect for Data

This is what data engineers value most:

```
DO:
✓ "This requires careful handling due to..."
✓ "Let me validate that assumption..."
✓ "How do we ensure data quality?"
✓ "What's our recovery plan if something breaks?"

DON'T:
✗ "The database will figure it out"
✗ "Let's just load all the data"
✗ "Performance doesn't matter for this"
✗ "We'll deal with data quality later"
```

---

## Final Interview Tips

1. **Before the interview:**
   - Review your most successful data-intensive projects
   - Prepare 30-min deep dives on 2-3 projects
   - Research their tech stack (their AWS services, databases)
   - Understand their scale ("How much data?", "How many daily users?")

2. **During the interview:**
   - Ask clarifying questions before diving into design
   - Draw pictures of architectures
   - Discuss trade-offs explicitly
   - Admit when you don't know, but show how you'd learn
   - Give concrete numbers (latency, throughput, cost)

3. **Technical Deep Dives:**
   - Be ready to code: Connection pooling, batch operations, pagination
   - Know your frameworks: Hibernate, Spring Data, Kafka
   - Understand your databases: Query plans, index strategies

4. **Testing Knowledge:**
   - How do you test with large datasets?
   - How do you simulate failures?
   - What monitoring do you put in place?

5. **The Closing:**
   - Ask: "How do you measure success for this role?"
   - Ask: "What's the biggest data challenge you're facing?"
   - Ask: "How does this role work with the data team?"

---

This interviewer will be most impressed by developers who:
- Think about data implications of every design decision
- Understand scale and have solved problems at scale
- Care deeply about data quality and reliability
- Ask intelligent questions about their data challenges
- Can articulate trade-offs between competing solutions
- Show respect for data as a core asset
