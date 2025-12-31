---
layout: post
title: "Kafka 재시도 로직과 Idempotency 패턴: 중복 메시지 처리 완전 가이드"
date: 2025-12-31
categories: [kafka, architecture, spring]
tags: [Kafka, 재시도, Retry, Idempotency, 멱등성, DLQ, DeadLetterQueue, SpringKafka, 중복처리]
---

# Kafka 재시도 로직과 Idempotency 패턴: 중복 메시지 처리 완전 가이드

이전 글에서 Outbox 패턴을 통해 DB 트랜잭션과 Kafka 메시지 발행 간의 일관성을 보장하는 방법을 다뤘습니다. 

하지만 Outbox 패턴을 사용하더라도, 또는 일반적인 Kafka 사용에서도 **재시도(Retry)로 인한 중복 메시지**는 피할 수 없습니다. 네트워크 오류, 브로커 장애, 타임아웃 등 다양한 이유로 메시지가 중복 발행되거나 중복 소비될 수 있습니다.

이번 글에서는 **Kafka의 재시도 로직**과 **Idempotency(멱등성) 패턴**을 통해 중복 메시지를 안전하게 처리하는 방법을 정리해보겠습니다.

---

## 1. 왜 중복 메시지가 발생하는가?

### 1.1 Producer 측 중복 발행

**시나리오 1: 네트워크 오류 후 재시도**

```java
// Producer가 메시지 발행
kafkaTemplate.send("order-created", event);

// 네트워크 오류 발생
// → Producer는 ACK를 받지 못함
// → 재시도 로직에 의해 동일한 메시지 재발행
// → 결과: 동일한 메시지가 2번 발행됨
```

**시나리오 2: 브로커 장애 후 재시도**

```
1. Producer가 메시지 발행 → 브로커에 도달
2. 브로커가 메시지를 저장 중 장애 발생
3. Producer는 ACK를 받지 못함
4. Producer가 재시도 → 브로커 복구 후 메시지 저장
5. 결과: 동일한 메시지가 2번 저장됨
```

### 1.2 Consumer 측 중복 소비

**시나리오 1: 처리 중 예외 발생**

```java
@KafkaListener(topics = "order-created")
public void handleOrderCreated(OrderCreatedEvent event) {
    // 1. 메시지 처리 시작
    processOrder(event);
    
    // 2. 예외 발생 (DB 연결 실패 등)
    throw new RuntimeException("DB connection failed");
    
    // 3. Consumer는 offset을 커밋하지 않음
    // 4. Consumer가 재시작되면 동일한 메시지 재소비
    // 결과: 동일한 메시지가 2번 처리됨
}
```

**시나리오 2: 처리 완료 후 커밋 실패**

```java
@KafkaListener(topics = "order-created")
public void handleOrderCreated(OrderCreatedEvent event) {
    // 1. 메시지 처리 완료
    processOrder(event);
    
    // 2. Offset 커밋 시도 → 네트워크 오류
    // 3. Consumer가 재시작되면 동일한 메시지 재소비
    // 결과: 동일한 메시지가 2번 처리됨
}
```

### 1.3 중복 메시지의 영향

**비즈니스 로직에 따라 심각한 문제 발생 가능:**

```java
// 주문 생성 이벤트
@KafkaListener(topics = "order-created")
public void handleOrderCreated(OrderCreatedEvent event) {
    // 중복 메시지가 들어오면?
    // → 주문이 2번 생성됨
    // → 재고가 2번 차감됨
    // → 결제가 2번 발생함
    orderService.createOrder(event);
    inventoryService.decreaseStock(event.getProductId(), event.getQuantity());
    paymentService.processPayment(event.getOrderId(), event.getAmount());
}
```

---

## 2. Kafka Producer 재시도 설정

### 2.1 기본 재시도 설정

**application.yml:**

```yaml
spring:
  kafka:
    producer:
      bootstrap-servers: localhost:9092
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      
      # 재시도 설정
      retries: 3  # 최대 재시도 횟수
      retry-backoff-ms: 100  # 재시도 간격 (100ms)
      
      # Idempotent Producer 설정
      enable-idempotence: true  # 중복 발행 방지
      acks: all  # 모든 리더와 팔로워가 메시지 수신 확인
      max-in-flight-requests-per-connection: 5  # 동시 요청 수 제한
```

**설정 설명:**

- **`retries`**: 메시지 발행 실패 시 최대 재시도 횟수
- **`retry-backoff-ms`**: 재시도 간격 (밀리초)
- **`enable-idempotence: true`**: **Idempotent Producer 활성화**
  - Producer가 동일한 메시지를 여러 번 발행해도 브로커에 한 번만 저장
  - Producer ID와 Sequence Number를 사용하여 중복 감지
- **`acks: all`**: 모든 리플리카가 메시지를 수신할 때까지 대기
- **`max-in-flight-requests-per-connection`**: 동시에 처리할 수 있는 요청 수

### 2.2 Idempotent Producer의 동작 원리

**Producer ID (PID)와 Sequence Number:**

```
1. Producer가 브로커에 연결
2. 브로커가 고유한 Producer ID (PID) 할당
3. Producer가 메시지 발행 시 Sequence Number 부여
   - 메시지 1: PID=123, Seq=1
   - 메시지 2: PID=123, Seq=2
   - 메시지 3: PID=123, Seq=3

4. 브로커가 중복 감지
   - 동일한 PID + Seq 조합이 이미 있으면 무시
   - 새로운 Seq면 저장
```

**중복 발행 방지:**

```java
// 첫 번째 발행
kafkaTemplate.send("order-created", event);  // PID=123, Seq=1

// 네트워크 오류로 ACK 미수신
// → 재시도
kafkaTemplate.send("order-created", event);  // PID=123, Seq=1 (동일)

// 브로커가 중복 감지 → 무시
// → 결과: 메시지가 1번만 저장됨
```

**주의사항:**

- **Idempotent Producer는 브로커 레벨에서만 중복 방지**
- **Consumer 측 중복 소비는 여전히 발생 가능**
- **여러 Producer 인스턴스가 동일한 메시지를 발행하면 중복 가능**

### 2.3 Exponential Backoff 재시도

**고정 간격 재시도의 문제:**

```yaml
spring:
  kafka:
    producer:
      retries: 3
      retry-backoff-ms: 100  # 항상 100ms 간격
```

**문제점:**
- 브로커가 일시적으로 과부하 상태일 때, 짧은 간격으로 재시도하면 부하 증가
- 여러 Producer가 동시에 재시도하면 **Thundering Herd 문제** 발생

**Exponential Backoff 적용:**

```java
@Configuration
public class KafkaProducerConfig {
    
    @Bean
    public ProducerFactory<String, Object> producerFactory() {
        Map<String, Object> configProps = new HashMap<>();
        configProps.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        configProps.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        configProps.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
        
        // 재시도 설정
        configProps.put(ProducerConfig.RETRIES_CONFIG, 5);
        configProps.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
        configProps.put(ProducerConfig.ACKS_CONFIG, "all");
        
        // Exponential Backoff를 위한 커스텀 RetryBackoffPolicy
        // (Spring Kafka는 기본적으로 Exponential Backoff를 지원하지 않음)
        // → 커스텀 ProducerInterceptor 구현 필요
        
        return new DefaultKafkaProducerFactory<>(configProps);
    }
}
```

**커스텀 Retry 로직:**

```java
@Component
@Slf4j
public class RetryableKafkaProducer {
    
    private final KafkaTemplate<String, Object> kafkaTemplate;
    
    public void sendWithRetry(String topic, String key, Object message) {
        int maxRetries = 5;
        long initialDelay = 100;  // 100ms
        long maxDelay = 5000;     // 5초
        
        for (int attempt = 0; attempt < maxRetries; attempt++) {
            try {
                kafkaTemplate.send(topic, key, message).get();
                return;  // 성공 시 종료
            } catch (Exception e) {
                if (attempt == maxRetries - 1) {
                    throw new RuntimeException("Failed to send message after " + maxRetries + " retries", e);
                }
                
                // Exponential Backoff: 100ms → 200ms → 400ms → 800ms → 1600ms
                long delay = Math.min(initialDelay * (1L << attempt), maxDelay);
                log.warn("Failed to send message, retrying in {}ms (attempt {}/{})", 
                    delay, attempt + 1, maxRetries);
                
                try {
                    Thread.sleep(delay);
                } catch (InterruptedException ie) {
                    Thread.currentThread().interrupt();
                    throw new RuntimeException("Interrupted during retry", ie);
                }
            }
        }
    }
}
```

---

## 3. Kafka Consumer 재시도 전략

### 3.1 Spring Kafka의 기본 재시도

**application.yml:**

```yaml
spring:
  kafka:
    consumer:
      bootstrap-servers: localhost:9092
      group-id: order-service
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      
      # Consumer 재시도 설정
      enable-auto-commit: false  # 수동 커밋 (재시도 제어를 위해)
      auto-offset-reset: earliest
      
    listener:
      # Listener 재시도 설정
      ack-mode: manual_immediate  # 수동 커밋 모드
      
      # 에러 핸들러
      type: batch  # 배치 모드 또는 record 모드
```

### 3.2 @RetryableTopic을 사용한 재시도

**Spring Kafka 2.7+ 부터 지원:**

```java
@Configuration
@EnableKafka
public class KafkaConfig {
    
    @Bean
    public KafkaListenerContainerFactory<ConcurrentMessageListenerContainer<String, Object>> 
        kafkaListenerContainerFactory() {
        ConcurrentKafkaListenerContainerFactory<String, Object> factory = 
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory());
        return factory;
    }
}
```

**@RetryableTopic 사용:**

```java
@Component
@Slf4j
public class OrderEventConsumer {
    
    @RetryableTopic(
        attempts = "4",  // 최대 4번 재시도
        backoff = @Backoff(delay = 1000, multiplier = 2),  // 1초 → 2초 → 4초 → 8초
        dltStrategy = DltStrategy.FAIL_ON_ERROR,  // 최종 실패 시 DLQ로 전송
        include = {RuntimeException.class},  // 재시도할 예외
        exclude = {IllegalArgumentException.class},  // 재시도하지 않을 예외
        autoCreateTopics = "true",  // 자동으로 재시도 토픽 생성
        topicSuffixingStrategy = TopicSuffixingStrategy.SUFFIX_WITH_INDEX_VALUE
    )
    @KafkaListener(topics = "order-created")
    public void handleOrderCreated(OrderCreatedEvent event) {
        log.info("Processing order created event: {}", event);
        
        try {
            orderService.createOrder(event);
            inventoryService.decreaseStock(event.getProductId(), event.getQuantity());
            paymentService.processPayment(event.getOrderId(), event.getAmount());
        } catch (Exception e) {
            log.error("Failed to process order created event: {}", event, e);
            throw e;  // 예외를 다시 던져서 재시도 트리거
        }
    }
}
```

**동작 방식:**

```
1. order-created 토픽에서 메시지 수신
2. 처리 실패 (예외 발생)
3. order-created-retry-0 토픽으로 메시지 이동 (1초 후)
4. 재처리 실패
5. order-created-retry-1 토픽으로 메시지 이동 (2초 후)
6. 재처리 실패
7. order-created-retry-2 토픽으로 메시지 이동 (4초 후)
8. 재처리 실패
9. order-created-retry-3 토픽으로 메시지 이동 (8초 후)
10. 재처리 실패
11. order-created-dlt 토픽으로 메시지 이동 (Dead Letter Queue)
```

**재시도 토픽 자동 생성:**

```yaml
spring:
  kafka:
    admin:
      properties:
        bootstrap.servers: localhost:9092
```

### 3.3 수동 재시도 구현

**@RetryableTopic을 사용하지 않는 경우:**

```java
@Component
@Slf4j
public class OrderEventConsumer {
    
    private final KafkaTemplate<String, Object> kafkaTemplate;
    private final ObjectMapper objectMapper;
    
    @KafkaListener(topics = "order-created")
    public void handleOrderCreated(
            ConsumerRecord<String, String> record,
            Acknowledgment acknowledgment) {
        
        try {
            OrderCreatedEvent event = objectMapper.readValue(
                record.value(), 
                OrderCreatedEvent.class
            );
            
            // 메시지 처리
            processOrder(event);
            
            // 성공 시 offset 커밋
            acknowledgment.acknowledge();
            
        } catch (RetryableException e) {
            // 재시도 가능한 예외
            log.warn("Retryable error occurred, will retry: {}", e.getMessage());
            
            // 재시도 토픽으로 전송
            sendToRetryTopic(record, e);
            
            // 원본 토픽의 offset은 커밋하지 않음 (재시도 토픽에서 처리)
            acknowledgment.acknowledge();
            
        } catch (NonRetryableException e) {
            // 재시도 불가능한 예외 (비즈니스 로직 오류 등)
            log.error("Non-retryable error occurred: {}", e.getMessage());
            
            // DLQ로 전송
            sendToDLQ(record, e);
            
            // offset 커밋 (더 이상 재시도하지 않음)
            acknowledgment.acknowledge();
        }
    }
    
    private void sendToRetryTopic(ConsumerRecord<String, String> record, Exception e) {
        RetryMessage retryMessage = RetryMessage.builder()
            .originalTopic(record.topic())
            .originalPartition(record.partition())
            .originalOffset(record.offset())
            .payload(record.value())
            .retryCount(0)
            .errorMessage(e.getMessage())
            .timestamp(LocalDateTime.now())
            .build();
        
        // 재시도 토픽으로 전송 (지연 발행)
        kafkaTemplate.send("order-created-retry", record.key(), retryMessage);
    }
}
```

### 3.4 Dead Letter Queue (DLQ) 패턴

**DLQ란?**

- 최대 재시도 횟수를 초과한 메시지를 저장하는 특별한 토픽
- 수동 개입이 필요한 메시지를 별도로 관리
- 운영자가 문제를 분석하고 수동으로 재처리 가능

**DLQ 구현:**

```java
@Component
@Slf4j
public class DLQHandler {
    
    private final KafkaTemplate<String, Object> kafkaTemplate;
    private final DLQRepository dlqRepository;
    
    public void sendToDLQ(String originalTopic, String key, String value, Exception error) {
        DLQMessage dlqMessage = DLQMessage.builder()
            .originalTopic(originalTopic)
            .key(key)
            .payload(value)
            .errorMessage(error.getMessage())
            .errorStacktrace(getStackTrace(error))
            .timestamp(LocalDateTime.now())
            .build();
        
        // DLQ 토픽으로 전송
        kafkaTemplate.send("dlq-" + originalTopic, key, dlqMessage);
        
        // DB에도 저장 (조회 및 분석 용이)
        dlqRepository.save(dlqMessage);
    }
    
    // 운영자가 DLQ 메시지를 재처리하는 API
    @Transactional
    public void reprocessDLQMessage(Long dlqMessageId) {
        DLQMessage dlqMessage = dlqRepository.findById(dlqMessageId)
            .orElseThrow(() -> new IllegalArgumentException("DLQ message not found"));
        
        try {
            // 원본 토픽으로 재전송
            kafkaTemplate.send(
                dlqMessage.getOriginalTopic(),
                dlqMessage.getKey(),
                dlqMessage.getPayload()
            );
            
            // DLQ에서 제거
            dlqRepository.delete(dlqMessage);
            
            log.info("DLQ message {} reprocessed successfully", dlqMessageId);
        } catch (Exception e) {
            log.error("Failed to reprocess DLQ message: {}", dlqMessageId, e);
            throw e;
        }
    }
}
```

---

## 4. Idempotency 패턴: 중복 메시지 처리

### 4.1 Idempotency란?

**멱등성(Idempotency):**
- 동일한 작업을 여러 번 수행해도 결과가 동일한 성질
- 예: `f(x) = f(f(x)) = f(f(f(x)))`

**Kafka에서의 Idempotency:**
- 동일한 메시지를 여러 번 처리해도 결과가 동일해야 함
- 중복 메시지가 들어와도 비즈니스 로직이 안전하게 처리

### 4.2 Idempotency Key 전략

**메시지에 고유 ID 포함:**

```java
public class OrderCreatedEvent {
    private String eventId;  // 고유 이벤트 ID (UUID)
    private Long orderId;
    private Long userId;
    private BigDecimal amount;
    private LocalDateTime createdAt;
}
```

**Consumer에서 중복 체크:**

```java
@Component
@Slf4j
public class OrderEventConsumer {
    
    private final Set<String> processedEventIds = ConcurrentHashMap.newKeySet();
    
    @KafkaListener(topics = "order-created")
    public void handleOrderCreated(OrderCreatedEvent event) {
        String eventId = event.getEventId();
        
        // 이미 처리한 이벤트인지 확인
        if (processedEventIds.contains(eventId)) {
            log.warn("Duplicate event detected and ignored: {}", eventId);
            return;  // 중복 이벤트 무시
        }
        
        // 이벤트 처리
        processOrder(event);
        
        // 처리 완료 표시
        processedEventIds.add(eventId);
    }
}
```

**문제점:**
- 메모리 기반 저장 → 서비스 재시작 시 정보 손실
- 메모리 사용량 증가 (이벤트가 많을수록)

### 4.3 DB 기반 Idempotency

**더 안정적인 방법: DB에 처리 이벤트 ID 저장**

```sql
CREATE TABLE processed_events (
    event_id VARCHAR(255) PRIMARY KEY,
    event_type VARCHAR(100) NOT NULL,
    processed_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_processed_at (processed_at)
);
```

**Consumer 구현:**

```java
@Component
@Slf4j
@Transactional
public class OrderEventConsumer {
    
    private final ProcessedEventRepository processedEventRepository;
    
    @KafkaListener(topics = "order-created")
    public void handleOrderCreated(OrderCreatedEvent event) {
        String eventId = event.getEventId();
        
        // 이미 처리한 이벤트인지 확인
        if (processedEventRepository.existsById(eventId)) {
            log.warn("Duplicate event detected and ignored: {}", eventId);
            return;  // 중복 이벤트 무시
        }
        
        // 이벤트 처리
        try {
            processOrder(event);
            
            // 처리 완료 기록
            ProcessedEvent processedEvent = ProcessedEvent.builder()
                .eventId(eventId)
                .eventType("OrderCreated")
                .processedAt(LocalDateTime.now())
                .build();
            processedEventRepository.save(processedEvent);
            
        } catch (Exception e) {
            log.error("Failed to process event: {}", eventId, e);
            throw e;  // 예외를 다시 던져서 재시도 트리거
        }
    }
}
```

**트랜잭션 고려사항:**

```java
@Transactional
public void handleOrderCreated(OrderCreatedEvent event) {
    // 1. 중복 체크
    if (processedEventRepository.existsById(event.getEventId())) {
        return;
    }
    
    // 2. 비즈니스 로직 처리
    processOrder(event);  // Order 테이블에 INSERT
    
    // 3. 처리 완료 기록
    processedEventRepository.save(new ProcessedEvent(event.getEventId()));
    
    // 4. 트랜잭션 커밋
    // → Order와 ProcessedEvent가 함께 저장됨 (원자적 보장)
}
```

### 4.4 Redis 기반 Idempotency

**TTL을 활용한 자동 정리:**

```java
@Component
@Slf4j
public class OrderEventConsumer {
    
    private final RedisTemplate<String, String> redisTemplate;
    private static final String PROCESSED_EVENTS_KEY = "processed:events:";
    private static final long TTL_SECONDS = 86400;  // 24시간
    
    @KafkaListener(topics = "order-created")
    public void handleOrderCreated(OrderCreatedEvent event) {
        String eventId = event.getEventId();
        String key = PROCESSED_EVENTS_KEY + eventId;
        
        // Redis에 이미 존재하는지 확인 (SETNX: Set if Not eXists)
        Boolean isNew = redisTemplate.opsForValue().setIfAbsent(key, "1", 
            Duration.ofSeconds(TTL_SECONDS));
        
        if (Boolean.FALSE.equals(isNew)) {
            log.warn("Duplicate event detected and ignored: {}", eventId);
            return;  // 중복 이벤트 무시
        }
        
        // 이벤트 처리
        processOrder(event);
    }
}
```

**장점:**
- 빠른 조회 성능 (메모리 기반)
- TTL로 자동 정리 (메모리 효율적)
- 분산 환경에서도 동작 (여러 Consumer 인스턴스 간 공유)

**단점:**
- Redis 장애 시 Idempotency 보장 불가
- 영구 저장이 필요한 경우 부적합

### 4.5 비즈니스 키 기반 Idempotency

**이벤트 ID가 없는 경우: 비즈니스 키 조합 사용**

```java
@Component
@Slf4j
public class OrderEventConsumer {
    
    private final ProcessedEventRepository processedEventRepository;
    
    @KafkaListener(topics = "order-created")
    public void handleOrderCreated(OrderCreatedEvent event) {
        // 비즈니스 키 조합: orderId + createdAt
        String idempotencyKey = generateIdempotencyKey(
            event.getOrderId(),
            event.getCreatedAt()
        );
        
        // 이미 처리한 이벤트인지 확인
        if (processedEventRepository.existsById(idempotencyKey)) {
            log.warn("Duplicate event detected and ignored: {}", idempotencyKey);
            return;
        }
        
        // 이벤트 처리
        processOrder(event);
        
        // 처리 완료 기록
        processedEventRepository.save(new ProcessedEvent(idempotencyKey));
    }
    
    private String generateIdempotencyKey(Long orderId, LocalDateTime createdAt) {
        return String.format("%d-%s", orderId, createdAt.toString());
    }
}
```

---

## 5. 실전 예제: 완전한 구현

### 5.1 전체 아키텍처

```java
// 1. 이벤트 정의
@Data
@Builder
public class OrderCreatedEvent {
    private String eventId;  // UUID
    private Long orderId;
    private Long userId;
    private BigDecimal amount;
    private LocalDateTime createdAt;
}

// 2. ProcessedEvent Entity
@Entity
@Table(name = "processed_events")
public class ProcessedEvent {
    @Id
    private String eventId;
    
    private String eventType;
    private LocalDateTime processedAt;
}

// 3. Consumer with Idempotency
@Component
@Slf4j
@Transactional
public class OrderEventConsumer {
    
    private final ProcessedEventRepository processedEventRepository;
    private final OrderService orderService;
    
    @RetryableTopic(
        attempts = "4",
        backoff = @Backoff(delay = 1000, multiplier = 2),
        dltStrategy = DltStrategy.FAIL_ON_ERROR,
        include = {RetryableException.class}
    )
    @KafkaListener(topics = "order-created")
    public void handleOrderCreated(OrderCreatedEvent event) {
        // Idempotency 체크
        if (processedEventRepository.existsById(event.getEventId())) {
            log.warn("Duplicate event ignored: {}", event.getEventId());
            return;
        }
        
        // 비즈니스 로직 처리
        try {
            orderService.createOrder(event);
            
            // 처리 완료 기록
            ProcessedEvent processedEvent = ProcessedEvent.builder()
                .eventId(event.getEventId())
                .eventType("OrderCreated")
                .processedAt(LocalDateTime.now())
                .build();
            processedEventRepository.save(processedEvent);
            
        } catch (NonRetryableException e) {
            // 비즈니스 로직 오류 → DLQ로 전송
            log.error("Non-retryable error: {}", e.getMessage());
            throw e;
        } catch (Exception e) {
            // 재시도 가능한 오류 → 재시도 트리거
            log.error("Retryable error: {}", e.getMessage());
            throw e;
        }
    }
}

// 4. DLQ Handler
@Component
@Slf4j
public class DLQHandler {
    
    private final KafkaTemplate<String, Object> kafkaTemplate;
    private final DLQRepository dlqRepository;
    
    @KafkaListener(topics = "order-created-dlt")
    public void handleDLQMessage(DLQMessage message) {
        log.error("DLQ message received: {}", message);
        
        // DB에 저장
        dlqRepository.save(message);
        
        // 알림 발송 (Slack, Email 등)
        alertService.sendAlert("DLQ message received", message);
    }
}
```

### 5.2 설정 파일

**application.yml:**

```yaml
spring:
  kafka:
    producer:
      bootstrap-servers: localhost:9092
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      retries: 3
      enable-idempotence: true
      acks: all
      max-in-flight-requests-per-connection: 5
      
    consumer:
      bootstrap-servers: localhost:9092
      group-id: order-service
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      enable-auto-commit: false
      auto-offset-reset: earliest
      
    listener:
      ack-mode: manual_immediate
      type: record
      
    admin:
      properties:
        bootstrap.servers: localhost:9092
```

---

## 6. 모니터링 및 관찰 가능성

### 6.1 재시도 메트릭

```java
@Component
@Slf4j
public class KafkaRetryMetrics {
    
    private final MeterRegistry meterRegistry;
    
    public void recordRetry(String topic, int retryCount) {
        meterRegistry.counter("kafka.retry.count", 
            "topic", topic,
            "retry_count", String.valueOf(retryCount)
        ).increment();
    }
    
    public void recordDLQMessage(String topic) {
        meterRegistry.counter("kafka.dlq.count", "topic", topic).increment();
    }
    
    public void recordDuplicateEvent(String eventType) {
        meterRegistry.counter("kafka.duplicate.event", "type", eventType).increment();
    }
}
```

### 6.2 로깅

```java
@Component
@Slf4j
public class OrderEventConsumer {
    
    @KafkaListener(topics = "order-created")
    public void handleOrderCreated(OrderCreatedEvent event) {
        MDC.put("eventId", event.getEventId());
        MDC.put("orderId", String.valueOf(event.getOrderId()));
        
        try {
            if (processedEventRepository.existsById(event.getEventId())) {
                log.warn("Duplicate event ignored");
                return;
            }
            
            processOrder(event);
            processedEventRepository.save(new ProcessedEvent(event.getEventId()));
            
            log.info("Event processed successfully");
        } catch (Exception e) {
            log.error("Failed to process event", e);
            throw e;
        } finally {
            MDC.clear();
        }
    }
}
```

---

## 7. 주의사항 및 Best Practices

### 7.1 Idempotency Key 선택

**✅ 좋은 Idempotency Key:**
- UUID (고유성 보장)
- 비즈니스 키 조합 (orderId + timestamp)
- 이벤트 소스에서 생성한 고유 ID

**❌ 나쁜 Idempotency Key:**
- 순차적 ID (중복 가능)
- 타임스탬프만 사용 (밀리초 단위 중복 가능)
- 비즈니스 키만 사용 (재처리 시 문제)

### 7.2 재시도 횟수와 간격

**권장 설정:**
- **재시도 횟수**: 3~5회
- **초기 지연**: 1초
- **최대 지연**: 30초
- **Exponential Backoff**: multiplier 2

**과도한 재시도 피하기:**
- 너무 많은 재시도 → 브로커 부하 증가
- 너무 짧은 간격 → Thundering Herd 문제

### 7.3 DLQ 모니터링

**DLQ 메시지가 쌓이면:**
1. 즉시 알림 발송
2. 원인 분석 (로그, 메트릭 확인)
3. 수동 재처리 또는 수정 후 재처리
4. 근본 원인 해결 (비즈니스 로직 수정 등)

### 7.4 ProcessedEvent 테이블 정리

**오래된 레코드 정리:**

```java
@Scheduled(cron = "0 0 2 * * *")  // 매일 새벽 2시
public void cleanupProcessedEvents() {
    LocalDateTime cutoffDate = LocalDateTime.now().minusDays(30);  // 30일 전
    
    processedEventRepository.deleteByProcessedAtBefore(cutoffDate);
}
```

---

## 마무리

**핵심 포인트:**

- **Kafka Producer의 Idempotent Producer 설정으로 브로커 레벨 중복 발행 방지 가능**
- **Consumer 측 중복 소비는 재시도로 인해 필연적으로 발생하므로 Idempotency 패턴 필수**
- **@RetryableTopic을 사용하면 재시도 토픽과 DLQ를 자동으로 관리할 수 있음**
- **DB 또는 Redis 기반 Idempotency 체크로 중복 메시지를 안전하게 처리**
- **DLQ를 통해 최종 실패한 메시지를 별도 관리하고 수동 재처리 가능**

Kafka를 사용한 분산 시스템에서 **재시도와 Idempotency는 필수적인 패턴**입니다. Outbox 패턴과 함께 사용하면 트랜잭션 일관성과 메시지 안정성을 모두 확보할 수 있습니다.

다음 글에서는 이러한 패턴들을 종합한 **실전 Kafka 마이크로서비스 아키텍처 설계**를 정리해볼 예정입니다. 🚀

