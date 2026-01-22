---
layout: post
title: "Kafka Producer 동기/비동기 성능 비교: Spring Kafka 3.3.11 기준 완전 정리"
date: 2026-01-22
categories: [kafka, performance, spring]
tags: [Kafka, Producer, 동기, 비동기, Synchronized, Asynchronized, 성능최적화, SpringKafka, KafkaTemplate, Throughput, Latency]
---

이전 글에서 Kubernetes IaC 도구들의 특징과 사용 상황을 비교했습니다. 이번 글에서는 **Kafka Producer의 동기/비동기 방식에 따른 성능 차이**를 Spring Kafka 3.3.11 기준으로 정리해보겠습니다.

Kafka Producer에서 메시지를 전송하는 방식은 **동기(Synchronous)**와 **비동기(Asynchronous)**로 나뉩니다. 실제 측정 결과, 동기와 비동기 방식 사이에는 **55~70배의 성능 차이**가 발생합니다. 이 차이를 이해하고 올바른 방식을 선택하는 것이 중요합니다.

---

## 1. Producer 동기/비동기란?

### 1.1 동기(Synchronous) 방식

**동기 방식의 특징:**

- 메시지 전송 후 **브로커의 응답을 기다림**
- 응답을 받은 후에만 다음 메시지 전송
- 전송 성공/실패를 **즉시 확인 가능**

**동작 흐름:**
```
Producer → Send Message 1 → Wait for Response → Success
         → Send Message 2 → Wait for Response → Success
         → Send Message 3 → Wait for Response → Success
```

**장점:**
- ✅ 전송 성공/실패 즉시 확인
- ✅ 에러 처리 간단
- ✅ 순서 보장 용이

**단점:**
- ❌ **성능 저하**: 응답 대기 시간만큼 지연
- ❌ **처리량 감소**: 순차 처리로 인한 병목

### 1.2 비동기(Asynchronous) 방식

**비동기 방식의 특징:**

- 메시지 전송 후 **응답을 기다리지 않음**
- 여러 메시지를 **동시에 전송** (batching)
- 콜백(Callback)으로 성공/실패 처리

**동작 흐름:**
```
Producer → Send Message 1 → (Continue immediately)
         → Send Message 2 → (Continue immediately)
         → Send Message 3 → (Continue immediately)
         
         → Callback: Message 1 Success
         → Callback: Message 2 Success
         → Callback: Message 3 Success
```

**장점:**
- ✅ **높은 처리량**: 여러 메시지 병렬 전송
- ✅ **낮은 지연 시간**: 응답 대기 시간 제거
- ✅ **리소스 효율**: 네트워크 I/O 최적화

**단점:**
- ❌ 에러 처리 복잡 (콜백 필요)
- ❌ 전송 실패 시 추적 어려움 (별도 DLQ 필요)

---

## 2. 성능 비교 데이터

### 2.1 측정 조건

**테스트 환경:**
- 메시지 개수: 10,808건
- Replication factor: 2
- 기타 설정:
  - `max.in.flight.requests.per.connection: 5`
  - `retries: 3`
  - `reconnect.backoff.max.ms: 1000L`
  - `batch.size: 5KB`
  - `linger.ms: 10ms`
  - `compression.type: gzip`

### 2.2 성능 비교 결과

| 방식 | acks | Duration (ms) | Duration (분) | 처리 속도 (msg/sec) |
|------|------|---------------|---------------|---------------------|
| **동기** | all | 286,242 | 4.77 | 37.8 |
| **비동기** | all | 4,581 | 0.076 | 2,359.6 |
| **동기** | 1 | 247,904 | 4.13 | 43.6 |
| **비동기** | 1 | 1,149 | 0.019 | 9,405.8 |

**성능 순서:**
1. **acks=1 비동기** (가장 빠름)
2. **acks=all 비동기**
3. **acks=1 동기**
4. **acks=all 동기** (가장 느림)

### 2.3 주요 발견 사항

**1. 동기/비동기 차이가 압도적:**
- 동기/비동기 차이: **약 55~70배**
- `acks` 옵션 차이: 약 **15%** (replica factor: 2 기준)

**2. 성능에 가장 큰 영향: 동기/비동기 여부**
- `acks` 설정보다 **동기/비동기 방식이 성능에 더 큰 영향**
- 1만 건 처리 시:
  - 동기: 약 **4~5분** 소요
  - 비동기: 약 **1~5초** 소요

**3. acks 옵션의 영향:**
- 동기 방식에서: acks=all이 약 **15% 느림**
- 비동기 방식에서: acks=all이 약 **75% 느림** (상대적으로 큰 차이)
- 하지만 **절대값으로는 비동기가 훨씬 빠름**

---

## 3. Spring Kafka 3.3.11에서의 구현

### 3.1 KafkaTemplate 기본 동작

**Spring Kafka의 `KafkaTemplate`은 기본적으로 비동기 방식입니다:**

```java
@Service
@Slf4j
public class OrderEventProducer {
    
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;
    
    // 기본 방식: 비동기 (즉시 반환)
    public void sendOrderEvent(OrderEvent event) {
        kafkaTemplate.send("order-created", event.getOrderId().toString(), event);
        // send() 메서드는 즉시 반환됨 (비동기)
    }
}
```

**동작 방식:**
- `send()` 메서드는 `ListenableFuture`를 반환하지만, 값을 사용하지 않으면 비동기로 동작
- 내부적으로 Producer의 배치 처리 및 비동기 전송 활용

### 3.2 동기 방식 구현

**동기 방식으로 전송하려면 `Future.get()`을 호출해야 합니다:**

```java
@Service
@Slf4j
public class OrderEventProducerSync {
    
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;
    
    // 동기 방식: 응답을 기다림
    public void sendOrderEventSync(OrderEvent event) {
        try {
            SendResult<String, Object> result = kafkaTemplate
                .send("order-created", event.getOrderId().toString(), event)
                .get(); // Future.get()으로 응답 대기 (동기)
            
            log.info("Message sent successfully: offset={}, partition={}", 
                result.getRecordMetadata().offset(),
                result.getRecordMetadata().partition());
        } catch (InterruptedException | ExecutionException e) {
            log.error("Failed to send message", e);
            throw new RuntimeException("Failed to send message", e);
        }
    }
    
    // 타임아웃 설정
    public void sendOrderEventSyncWithTimeout(OrderEvent event) {
        try {
            SendResult<String, Object> result = kafkaTemplate
                .send("order-created", event.getOrderId().toString(), event)
                .get(5, TimeUnit.SECONDS); // 5초 타임아웃
            
            log.info("Message sent: offset={}", 
                result.getRecordMetadata().offset());
        } catch (TimeoutException e) {
            log.error("Send timeout", e);
            throw new RuntimeException("Send timeout", e);
        } catch (InterruptedException | ExecutionException e) {
            log.error("Failed to send message", e);
            throw new RuntimeException("Failed to send message", e);
        }
    }
}
```

**특징:**
- `Future.get()` 호출 시 스레드가 블로킹됨
- 응답을 받을 때까지 대기
- 성능 저하 발생 (동기 방식)

### 3.3 비동기 방식 구현 (콜백 사용)

**비동기 방식에서 성공/실패 처리는 콜백으로 합니다:**

```java
@Service
@Slf4j
public class OrderEventProducerAsync {
    
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;
    
    // 비동기 방식: 콜백으로 처리
    public void sendOrderEventAsync(OrderEvent event) {
        ListenableFuture<SendResult<String, Object>> future = 
            kafkaTemplate.send("order-created", event.getOrderId().toString(), event);
        
        // 성공 콜백
        future.addCallback(
            result -> {
                log.info("Message sent successfully: offset={}, partition={}", 
                    result.getRecordMetadata().offset(),
                    result.getRecordMetadata().partition());
            },
            failure -> {
                log.error("Failed to send message: {}", failure.getMessage(), failure);
                // DLQ로 전송하거나 재시도 로직 추가
                handleFailure(event, failure);
            }
        );
    }
    
    // 더 세밀한 콜백 처리
    public void sendOrderEventAsyncDetailed(OrderEvent event) {
        kafkaTemplate.send("order-created", event.getOrderId().toString(), event)
            .addCallback(new ListenableFutureCallback<SendResult<String, Object>>() {
                
                @Override
                public void onSuccess(SendResult<String, Object> result) {
                    RecordMetadata metadata = result.getRecordMetadata();
                    log.info("Message sent successfully: topic={}, partition={}, offset={}", 
                        metadata.topic(),
                        metadata.partition(),
                        metadata.offset());
                    
                    // 성공 후 비즈니스 로직 (예: DB 업데이트)
                    updateOrderStatus(event.getOrderId(), "SENT");
                }
                
                @Override
                public void onFailure(Throwable ex) {
                    log.error("Failed to send message: orderId={}, error={}", 
                        event.getOrderId(), ex.getMessage(), ex);
                    
                    // 실패 처리 로직
                    handleFailure(event, ex);
                }
            });
    }
    
    private void handleFailure(OrderEvent event, Throwable failure) {
        // DLQ로 전송 또는 재시도 로직
        log.error("Handling send failure for orderId={}", event.getOrderId());
    }
    
    private void updateOrderStatus(String orderId, String status) {
        // DB 업데이트 로직
    }
}
```

**특징:**
- `send()` 메서드는 즉시 반환
- 콜백으로 성공/실패 처리
- 높은 처리량과 낮은 지연 시간

---

## 4. acks 옵션과 동기/비동기 조합

### 4.1 acks=1 비동기 (가장 빠름)

**설정:**
```yaml
spring:
  kafka:
    producer:
      acks: 1  # 리더만 확인
      properties:
        enable.idempotence: false
```

**Spring Kafka 설정:**
```java
@Configuration
public class KafkaProducerConfig {
    
    @Bean
    public ProducerFactory<String, Object> producerFactory() {
        Map<String, Object> configProps = new HashMap<>();
        configProps.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        configProps.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        configProps.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
        
        // 성능 최적화 설정
        configProps.put(ProducerConfig.ACKS_CONFIG, "1");  // 리더만 확인
        configProps.put(ProducerConfig.MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION, 5);
        configProps.put(ProducerConfig.RETRIES_CONFIG, 3);
        
        // Batch 설정
        configProps.put(ProducerConfig.BATCH_SIZE_CONFIG, 16384);  // 16KB
        configProps.put(ProducerConfig.LINGER_MS_CONFIG, 10);  // 10ms 대기
        configProps.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "gzip");
        
        return new DefaultKafkaProducerFactory<>(configProps);
    }
    
    @Bean
    public KafkaTemplate<String, Object> kafkaTemplate() {
        return new KafkaTemplate<>(producerFactory());
    }
}
```

**성능:**
- **가장 빠른 처리 속도**: 약 9,406 msg/sec
- 적절한 내구성 (리더에 쓰기 보장)
- 일반적인 사용 사례에 적합

### 4.2 acks=all 비동기 (내구성과 성능 균형)

**설정:**
```yaml
spring:
  kafka:
    producer:
      acks: all  # 리더 + 팔로워 확인
      properties:
        enable.idempotence: true
```

**Spring Kafka 설정:**
```java
@Configuration
public class KafkaProducerConfigDurable {
    
    @Bean
    public ProducerFactory<String, Object> producerFactory() {
        Map<String, Object> configProps = new HashMap<>();
        configProps.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        configProps.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        configProps.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
        
        // 내구성 최적화 설정
        configProps.put(ProducerConfig.ACKS_CONFIG, "all");  // 리더 + 팔로워
        configProps.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);  // 멱등성
        configProps.put(ProducerConfig.MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION, 5);
        configProps.put(ProducerConfig.RETRIES_CONFIG, 3);
        
        // Batch 설정 (성능 보완)
        configProps.put(ProducerConfig.BATCH_SIZE_CONFIG, 16384);  // 16KB
        configProps.put(ProducerConfig.LINGER_MS_CONFIG, 10);  // 10ms 대기
        configProps.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "gzip");
        
        return new DefaultKafkaProducerFactory<>(configProps);
    }
}
```

**성능:**
- acks=1 비동기 대비 약 **75% 느림** (상대적)
- 하지만 **동기 방식보다 62배 빠름** (절대적)
- 높은 내구성과 적절한 성능

### 4.3 enable.idempotence와 성능 Trade-off

**멱등성 보장 조건:**

1. `acks=all` 필수
2. `max.in.flight.requests.per.connection <= 5`
3. `retries > 0`

**성능 영향:**

- `enable.idempotence=true`는 약간의 성능 저하 발생
- 하지만 **동기/비동기 차이에 비하면 미미함**
- 내구성과 정확성을 위해 권장

**Kafka 3.0 이후 기본값:**

- Kafka 3.0부터는 `enable.idempotence=true`, `acks=all`이 **기본값**
- Kafka 3.3.11 사용 시 명시적으로 설정하지 않아도 자동 활성화

---

## 5. Batch 옵션으로 성능 보완

### 5.1 Batch 처리의 필요성

**`enable.idempotence=true` + `acks=all` 사용 시:**
- 성능 저하 발생
- **Batch 옵션으로 성능 보완 가능**

### 5.2 Batch 설정

**주요 옵션:**

1. **`batch.size`**: 배치 크기 (기본: 16KB)
2. **`linger.ms`**: 배치 대기 시간 (기본: 0ms)

**설정 예시:**
```java
@Configuration
public class KafkaProducerConfigBatch {
    
    @Bean
    public ProducerFactory<String, Object> producerFactory() {
        Map<String, Object> configProps = new HashMap<>();
        // ... 기본 설정 ...
        
        // Batch 최적화
        configProps.put(ProducerConfig.BATCH_SIZE_CONFIG, 32768);  // 32KB (더 큰 배치)
        configProps.put(ProducerConfig.LINGER_MS_CONFIG, 10);  // 10ms 대기 (배치 형성)
        configProps.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "gzip");  // 압축
        
        return new DefaultKafkaProducerFactory<>(configProps);
    }
}
```

**설정 전략:**

- **`batch.size`**:
  - 작은 값: 낮은 지연 시간, 낮은 처리량
  - 큰 값: 높은 처리량, 높은 지연 시간
  - 권장: 16KB ~ 64KB

- **`linger.ms`**:
  - 0ms: 배치 없이 즉시 전송 (낮은 처리량)
  - 10~100ms: 배치 형성 대기 (높은 처리량)
  - 권장: 10ms ~ 50ms

**주의사항:**

- Batch 크기가 너무 크면 메모리 사용량 증가
- `linger.ms`가 너무 길면 지연 시간 증가
- **성능 테스트를 통해 적정 값 찾기 필요**

### 5.3 application.yml 설정

```yaml
spring:
  kafka:
    producer:
      bootstrap-servers: localhost:9092
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      
      # acks 설정
      acks: all
      
      # 멱등성
      properties:
        enable.idempotence: true
      
      # 성능 최적화
      batch-size: 32768  # 32KB
      linger-ms: 10      # 10ms
      compression-type: gzip
      
      # 기타 설정
      max-in-flight-requests-per-connection: 5
      retries: 3
```

---

## 6. 실제 사용 시나리오별 권장 사항

### 6.1 단순 메시지 전송만 필요한 경우

**권장: acks=1 비동기 (기본 설정)**

```java
@Service
public class SimpleEventProducer {
    
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;
    
    // 기본 방식: 비동기, 즉시 반환
    public void sendEvent(Event event) {
        kafkaTemplate.send("topic-name", event.getId().toString(), event);
        // 성공/실패는 로그로만 처리
    }
}
```

**특징:**
- 가장 빠른 처리 속도
- Kafka Producer Default 설정 (Kafka 2.x 기준)
- 단순 이벤트 로깅, 메트릭 등에 적합

### 6.2 전송 결과에 따른 비즈니스 로직이 있는 경우

**권장: acks=all 비동기 + 콜백**

```java
@Service
@Slf4j
public class BusinessEventProducer {
    
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;
    
    @Autowired
    private OrderRepository orderRepository;
    
    // 비동기 + 콜백으로 전송 결과 처리
    public void sendOrderEvent(OrderEvent event) {
        kafkaTemplate.send("order-created", event.getOrderId().toString(), event)
            .addCallback(
                result -> {
                    // 성공 시: 주문 상태 업데이트
                    orderRepository.updateStatus(event.getOrderId(), "PROCESSING");
                    log.info("Order event sent: orderId={}", event.getOrderId());
                },
                failure -> {
                    // 실패 시: 주문 상태 롤백 또는 재시도
                    log.error("Failed to send order event: orderId={}", 
                        event.getOrderId(), failure);
                    handleSendFailure(event, failure);
                }
            );
    }
    
    private void handleSendFailure(OrderEvent event, Throwable failure) {
        // 실패 처리: 재시도 또는 DLQ 전송
    }
}
```

**특징:**
- 비동기로 높은 처리량 유지
- 콜백으로 전송 결과 처리
- 내구성 보장 (acks=all)

### 6.3 전송 실패 시 트랜잭션 롤백이 필요한 경우

**권장: 동기 방식 (필요시에만)**

```java
@Service
@Transactional
public class TransactionalEventProducer {
    
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;
    
    // 동기 방식: 전송 실패 시 트랜잭션 롤백
    public void createOrderWithEvent(Order order) {
        // 1. 주문 생성
        orderRepository.save(order);
        
        // 2. 이벤트 전송 (동기)
        try {
            kafkaTemplate.send("order-created", order.getId().toString(), 
                OrderEvent.from(order))
                .get(5, TimeUnit.SECONDS);  // 동기 대기
            
            // 성공 시: 커밋
        } catch (Exception e) {
            // 실패 시: 트랜잭션 롤백 (자동)
            throw new RuntimeException("Failed to send order event", e);
        }
    }
}
```

**주의사항:**
- 동기 방식은 **성능 저하가 큼** (55~70배 느림)
- **가능한 한 사용을 피하고**, 트랜잭션 보장이 정말 필요한 경우에만 사용
- 대안: Outbox 패턴 사용

### 6.4 높은 처리량이 필요한 경우

**권장: acks=1 비동기 + Batch 최적화**

```java
@Configuration
public class HighThroughputProducerConfig {
    
    @Bean
    public ProducerFactory<String, Object> producerFactory() {
        Map<String, Object> configProps = new HashMap<>();
        // ... 기본 설정 ...
        
        // 성능 최적화
        configProps.put(ProducerConfig.ACKS_CONFIG, "1");  // 빠른 응답
        configProps.put(ProducerConfig.BATCH_SIZE_CONFIG, 65536);  // 64KB (큰 배치)
        configProps.put(ProducerConfig.LINGER_MS_CONFIG, 20);  // 20ms 대기
        configProps.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "lz4");  // 빠른 압축
        configProps.put(ProducerConfig.MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION, 5);
        
        return new DefaultKafkaProducerFactory<>(configProps);
    }
}
```

**특징:**
- 최대 처리량 달성
- 약간의 데이터 유실 가능성 감수
- 로그, 메트릭 등에 적합

---

## 7. 성능 최적화 Best Practice

### 7.1 DO (해야 할 것)

1. **비동기 방식 기본 사용**
   ```java
   // ✅ 비동기 (권장)
   kafkaTemplate.send("topic", key, message);
   ```

2. **Batch 옵션 활용**
   ```yaml
   batch-size: 32768
   linger-ms: 10
   compression-type: gzip
   ```

3. **콜백으로 에러 처리**
   ```java
   kafkaTemplate.send("topic", key, message)
       .addCallback(success -> {}, failure -> {});
   ```

4. **토픽별로 다른 설정 사용**
   - 중요한 토픽: `acks=all` + `enable.idempotence=true`
   - 일반 토픽: `acks=1` + 비동기

### 7.2 DON'T (하지 말아야 할 것)

1. **동기 방식 남용 금지**
   ```java
   // ❌ 동기 방식 (성능 저하)
   kafkaTemplate.send("topic", key, message).get();
   ```

2. **불필요한 동기 대기 금지**
   ```java
   // ❌ 전송 결과를 기다릴 필요가 없는 경우
   SendResult result = kafkaTemplate.send(...).get();
   ```

3. **Batch 설정 무시 금지**
   - `linger.ms=0`과 `batch.size`를 함께 설정하지 않기

---

## 8. 실제 측정 예시

### 8.1 성능 테스트 코드

```java
@SpringBootTest
@Slf4j
public class KafkaProducerPerformanceTest {
    
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;
    
    @Test
    public void testAsyncPerformance() {
        int messageCount = 10000;
        long startTime = System.currentTimeMillis();
        
        // 비동기 전송
        CountDownLatch latch = new CountDownLatch(messageCount);
        for (int i = 0; i < messageCount; i++) {
            kafkaTemplate.send("test-topic", String.valueOf(i), 
                new TestEvent(i))
                .addCallback(
                    result -> latch.countDown(),
                    failure -> {
                        log.error("Failed", failure);
                        latch.countDown();
                    }
                );
        }
        
        // 모든 메시지 전송 완료 대기
        try {
            latch.await(60, TimeUnit.SECONDS);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        
        long duration = System.currentTimeMillis() - startTime;
        log.info("Async performance: {} messages in {} ms ({} msg/sec)", 
            messageCount, duration, messageCount * 1000.0 / duration);
    }
    
    @Test
    public void testSyncPerformance() {
        int messageCount = 1000;  // 동기는 더 적은 수로 테스트
        long startTime = System.currentTimeMillis();
        
        // 동기 전송
        for (int i = 0; i < messageCount; i++) {
            try {
                kafkaTemplate.send("test-topic", String.valueOf(i), 
                    new TestEvent(i)).get();
            } catch (Exception e) {
                log.error("Failed", e);
            }
        }
        
        long duration = System.currentTimeMillis() - startTime;
        log.info("Sync performance: {} messages in {} ms ({} msg/sec)", 
            messageCount, duration, messageCount * 1000.0 / duration);
    }
}
```

### 8.2 예상 결과

**비동기 (acks=1):**
- 10,000건: 약 1~2초
- 처리량: 약 5,000~10,000 msg/sec

**동기 (acks=1):**
- 1,000건: 약 20~25초
- 처리량: 약 40~50 msg/sec

**비동기가 동기보다 약 50~200배 빠름**

---

## 마무리

**핵심 포인트:**

1. **동기/비동기 차이가 압도적**: 약 55~70배 성능 차이
2. **Spring Kafka는 기본적으로 비동기**: `KafkaTemplate.send()`는 즉시 반환
3. **동기 방식은 필요시에만**: 트랜잭션 보장이 정말 필요한 경우
4. **Batch 옵션으로 성능 보완**: `enable.idempotence=true` + `acks=all` 사용 시 필수
5. **토픽별로 다른 전략**: 중요한 토픽은 내구성, 일반 토픽은 성능 우선

**권장 사항:**

- **대부분의 경우**: 비동기 방식 사용 (기본 설정)
- **전송 결과 확인 필요**: 비동기 + 콜백
- **트랜잭션 보장 필요**: 동기 방식 (또는 Outbox 패턴)
- **높은 처리량 필요**: acks=1 비동기 + Batch 최적화

**Kafka Producer Default 설정 (Kafka 2.x 기준):**
- `acks=1` + 비동기 = **가장 빠른 방식**
- 단순 메시지 전송만 한다면 기본 설정으로 충분

하지만 **전송 이후 메시지 결과에 따른 비즈니스 로직**이 있거나, **DB에 전송 결과를 저장**해야 하는 경우에는 Producer 설정을 변경하고 성능 체크를 해야 합니다. 가장 좋은 방법은 **전송된 메시지의 콜백을 받지 않아도 되는 구조로 설계**하는 것이지만, 콜백이 필요한 구조라면 최적의 Producer 설정을 찾아야 합니다. 🚀

다음 글에서는 **Kafka Consumer의 동기/비동기 처리 방식과 성능 최적화**를 Spring Kafka 3.3.11 기준으로 정리해보겠습니다.
