---
layout: post
title: "Kafka Consumer 동기/비동기 처리 방식: Spring Kafka 3.3.11 기준 성능 비교"
date: 2026-01-23
categories: [kafka, performance, spring]
tags: [Kafka, Consumer, 동기, 비동기, Polling, KafkaListener, 배치처리, Concurrency, SpringKafka, 성능최적화]
---

이전 글에서 Kafka Producer의 동기/비동기 성능 차이를 비교했습니다. 이번 글에서는 **Kafka Consumer의 동기/비동기 처리 방식과 성능 최적화**를 Spring Kafka 3.3.11 기준으로 정리해보겠습니다.

Kafka Consumer는 Producer와 달리 **메시지를 가져오는 방식**과 **메시지를 처리하는 방식**이 다릅니다. Spring Kafka의 `@KafkaListener`는 기본적으로 비동기 방식으로 동작하지만, 수동 Polling 방식과 비교했을 때 성능과 제어 방식에 차이가 있습니다.

---

## 1. Consumer 동기/비동기란?

### 1.1 Producer vs Consumer의 차이

**Producer (프로듀서):**
- 메시지를 **전송**하는 역할
- 동기: 전송 후 응답 대기
- 비동기: 전송 후 즉시 반환 (콜백)

**Consumer (컨슈머):**
- 메시지를 **수신 및 처리**하는 역할
- 동기 (Polling): 메시지를 가져와서 처리 완료 후 다음 메시지
- 비동기 (Listener): 메시지 수신과 처리가 분리되어 병렬 처리

### 1.2 Consumer의 두 가지 처리 방식

#### 방식 1: 수동 Polling (동기적)

**특징:**
- Consumer가 **직접 메시지를 가져옴** (`poll()`)
- 메시지 처리 완료 후 **다음 메시지 가져오기**
- **순차 처리**: 한 번에 하나씩 처리

**동작 흐름:**
```
Consumer → poll() → 메시지 가져오기
        → 메시지 처리 (동기)
        → 처리 완료
        → poll() → 다음 메시지 가져오기
```

#### 방식 2: Listener 패턴 (비동기적)

**특징:**
- Spring Kafka가 **내부적으로 Polling**
- 메시지 도착 시 **리스너 메서드 자동 호출**
- **병렬 처리 가능**: 여러 스레드가 동시에 처리

**동작 흐름:**
```
Spring Kafka → 내부 Polling (백그라운드)
             → 메시지 도착 시 리스너 호출
             → 리스너에서 메시지 처리 (비동기)
             → 처리 중에도 다음 메시지 Polling 계속
```

---

## 2. Spring Kafka의 @KafkaListener

### 2.1 @KafkaListener 기본 동작

**Spring Kafka의 `@KafkaListener`는 기본적으로 비동기 방식입니다:**

```java
@Component
@Slf4j
public class OrderConsumer {
    
    @KafkaListener(topics = "order-created")
    public void handleOrderCreated(OrderCreatedEvent event) {
        // 메시지 처리
        log.info("Processing order: {}", event.getOrderId());
        orderService.processOrder(event);
    }
}
```

**내부 동작:**
1. Spring Kafka가 내부적으로 `KafkaConsumer`를 생성
2. 백그라운드 스레드에서 **지속적으로 Polling**
3. 메시지 도착 시 리스너 메서드 호출
4. 처리 중에도 다음 메시지 계속 Polling

**특징:**
- ✅ **비동기 처리**: 메시지 수신과 처리가 분리
- ✅ **자동 Polling**: 개발자가 직접 `poll()` 호출 불필요
- ✅ **병렬 처리**: `concurrency` 옵션으로 병렬 처리 가능

### 2.2 @KafkaListener의 내부 구조

**Spring Kafka는 내부적으로 다음과 같이 동작합니다:**

```java
// Spring Kafka 내부 (의사 코드)
public class KafkaMessageListenerContainer {
    private final KafkaConsumer consumer;
    private final Executor executor;
    
    public void start() {
        executor.execute(() -> {
            while (running) {
                // Polling (백그라운드)
                ConsumerRecords records = consumer.poll(Duration.ofMillis(100));
                
                for (ConsumerRecord record : records) {
                    // 리스너 메서드 호출 (비동기)
                    listener.onMessage(record.value());
                }
            }
        });
    }
}
```

**실제로는:**
- `ConcurrentMessageListenerContainer`가 여러 스레드에서 Polling
- `@KafkaListener` 메서드는 별도 스레드 풀에서 실행
- **메시지 수신과 처리가 분리되어 비동기로 동작**

---

## 3. 수동 Polling 방식 (동기적)

### 3.1 수동 Polling 구현

**Spring Kafka에서 수동 Polling 방식:**

```java
@Service
@Slf4j
public class ManualPollingConsumer {
    
    @Autowired
    private KafkaConsumer<String, OrderCreatedEvent> kafkaConsumer;
    
    // 수동 Polling (동기적)
    public void consumeManually() {
        // Consumer 시작
        kafkaConsumer.subscribe(Collections.singletonList("order-created"));
        
        try {
            while (true) {
                // 메시지 가져오기 (동기 대기)
                ConsumerRecords<String, OrderCreatedEvent> records = 
                    kafkaConsumer.poll(Duration.ofMillis(100));
                
                // 메시지 처리 (순차)
                for (ConsumerRecord<String, OrderCreatedEvent> record : records) {
                    log.info("Processing message: offset={}, value={}", 
                        record.offset(), record.value());
                    
                    // 메시지 처리
                    processMessage(record.value());
                    
                    // Offset 커밋
                    kafkaConsumer.commitSync();
                }
            }
        } finally {
            kafkaConsumer.close();
        }
    }
    
    private void processMessage(OrderCreatedEvent event) {
        // 비즈니스 로직
        orderService.processOrder(event);
    }
}
```

**특징:**
- ✅ **완전한 제어**: Polling 시점, 처리 방식 직접 제어
- ✅ **순서 보장**: 메시지를 순차적으로 처리
- ❌ **성능 제한**: 한 번에 하나씩만 처리
- ❌ **복잡성**: Polling, 커밋 등을 직접 관리

### 3.2 수동 Polling 설정

**Consumer 설정:**

```java
@Configuration
public class KafkaConsumerConfig {
    
    @Bean
    public ConsumerFactory<String, Object> consumerFactory() {
        Map<String, Object> configProps = new HashMap<>();
        configProps.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        configProps.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        configProps.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, JsonDeserializer.class);
        configProps.put(ConsumerConfig.GROUP_ID_CONFIG, "my-group");
        configProps.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        
        // 수동 Polling 설정
        configProps.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);  // 수동 커밋
        
        return new DefaultKafkaConsumerFactory<>(configProps);
    }
    
    @Bean
    public KafkaConsumer<String, Object> kafkaConsumer() {
        return new KafkaConsumer<>(consumerFactory().getConfigurationProperties());
    }
}
```

**설정 파일 (application.yml):**

```yaml
spring:
  kafka:
    consumer:
      bootstrap-servers: localhost:9092
      group-id: my-group
      auto-offset-reset: earliest
      enable-auto-commit: false  # 수동 커밋
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
```

---

## 4. @KafkaListener vs 수동 Polling 성능 비교

### 4.1 처리 방식 비교

| 항목 | @KafkaListener (비동기) | 수동 Polling (동기) |
|------|------------------------|-------------------|
| **처리 방식** | 리스너 패턴 (자동) | 직접 Polling (수동) |
| **병렬 처리** | ✅ 가능 (concurrency) | ❌ 어려움 |
| **순서 보장** | ❌ 제한적 | ✅ 가능 |
| **성능** | 높음 | 낮음 |
| **제어** | 제한적 | 완전한 제어 |
| **복잡성** | 낮음 | 높음 |

### 4.2 성능 비교 예시

**@KafkaListener (비동기, concurrency=5):**
```
메시지 10,000건 처리:
- 처리 시간: 약 2초
- 처리량: 약 5,000 msg/sec
- 여러 스레드가 동시에 처리
```

**수동 Polling (동기):**
```
메시지 10,000건 처리:
- 처리 시간: 약 100초
- 처리량: 약 100 msg/sec
- 순차적으로 하나씩 처리
```

**@KafkaListener가 약 50배 빠름**

---

## 5. @KafkaListener 성능 최적화

### 5.1 Concurrency 설정

**Concurrency는 동시에 처리할 스레드 수를 설정합니다:**

```java
@Component
@Slf4j
public class OrderConsumer {
    
    // Concurrency 설정 (5개 스레드)
    @KafkaListener(topics = "order-created", concurrency = "5")
    public void handleOrderCreated(OrderCreatedEvent event) {
        log.info("Processing order: {}", event.getOrderId());
        orderService.processOrder(event);
    }
}
```

**설정 파일 (application.yml):**

```yaml
spring:
  kafka:
    listener:
      concurrency: 5  # 5개 스레드
```

**Concurrency 설정 전략:**

- **Partition 수와 일치**: 각 파티션당 1개 스레드
- **예: 파티션 3개 → concurrency=3**
- **파티션보다 많이 설정하면**: 일부 스레드가 유휴 상태
- **파티션보다 적게 설정하면**: 처리량 감소

**권장:**
```java
// 파티션 수와 동일하게 설정
@KafkaListener(topics = "order-created", concurrency = "3")  // 파티션 3개인 경우
```

### 5.2 배치 처리 (Batch Processing)

**여러 메시지를 한 번에 처리하여 성능 향상:**

```java
@Component
@Slf4j
public class BatchOrderConsumer {
    
    // 배치 처리 (여러 메시지를 한 번에 받음)
    @KafkaListener(topics = "order-created", 
                   containerFactory = "batchKafkaListenerContainerFactory")
    public void handleOrderCreatedBatch(
            List<ConsumerRecord<String, OrderCreatedEvent>> records) {
        
        log.info("Received {} messages", records.size());
        
        // 배치 처리
        for (ConsumerRecord<String, OrderCreatedEvent> record : records) {
            orderService.processOrder(record.value());
        }
        
        // 또는 DB 배치 처리
        List<OrderCreatedEvent> events = records.stream()
            .map(ConsumerRecord::value)
            .collect(Collectors.toList());
        
        orderService.processOrdersBatch(events);
    }
}
```

**Batch Listener Container Factory 설정:**

```java
@Configuration
public class KafkaBatchConfig {
    
    @Bean
    public ConsumerFactory<String, Object> consumerFactory() {
        Map<String, Object> configProps = new HashMap<>();
        configProps.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        configProps.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        configProps.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, JsonDeserializer.class);
        configProps.put(ConsumerConfig.GROUP_ID_CONFIG, "my-group");
        
        // 배치 처리 설정
        configProps.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, 500);  // 한 번에 최대 500개
        
        return new DefaultKafkaConsumerFactory<>(configProps);
    }
    
    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, Object> 
            batchKafkaListenerContainerFactory() {
        
        ConcurrentKafkaListenerContainerFactory<String, Object> factory =
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory());
        factory.setBatchListener(true);  // 배치 리스너 활성화
        
        return factory;
    }
}
```

**설정 파일 (application.yml):**

```yaml
spring:
  kafka:
    consumer:
      max-poll-records: 500  # 한 번에 최대 500개 메시지
    listener:
      type: batch  # 배치 리스너 활성화
```

### 5.3 성능 비교: 단일 처리 vs 배치 처리

**단일 처리 (@KafkaListener 기본):**
```
메시지 10,000건 처리:
- 처리 시간: 약 20초
- DB 쿼리: 10,000번
- 처리량: 약 500 msg/sec
```

**배치 처리 (max-poll-records=500):**
```
메시지 10,000건 처리:
- 처리 시간: 약 2초
- DB 쿼리: 20번 (배치로 500개씩)
- 처리량: 약 5,000 msg/sec
```

**배치 처리가 약 10배 빠름**

---

## 6. 순서 보장과 성능 Trade-off

### 6.1 순서 보장이 필요한 경우

**순서가 중요한 메시지 예시:**

```java
// 주문 상태 변경 이벤트
// 1. 주문 생성
// 2. 결제 완료
// 3. 배송 시작

// 순서가 보장되지 않으면?
// → 결제 완료가 주문 생성보다 먼저 처리될 수 있음
// → 비즈니스 로직 오류 발생
```

**순서 보장 설정:**

```java
@Component
@Slf4j
public class OrderedOrderConsumer {
    
    // Concurrency=1로 설정하여 순서 보장
    @KafkaListener(topics = "order-events", concurrency = "1")
    public void handleOrderEvent(OrderEvent event) {
        // 순서가 보장되어 처리됨
        orderService.processOrderEvent(event);
    }
}
```

**특징:**
- ✅ **순서 보장**: 메시지가 순차적으로 처리
- ❌ **성능 저하**: 한 번에 하나씩만 처리
- ❌ **처리량 감소**: 병렬 처리 불가

### 6.2 순서가 중요하지 않은 경우

**성능 최적화 설정:**

```java
@Component
@Slf4j
public class HighPerformanceOrderConsumer {
    
    // 높은 Concurrency로 병렬 처리
    @KafkaListener(topics = "order-created", concurrency = "10")
    public void handleOrderCreated(OrderCreatedEvent event) {
        // 병렬 처리로 높은 처리량
        orderService.processOrder(event);
    }
}
```

**특징:**
- ✅ **높은 처리량**: 여러 스레드가 동시에 처리
- ✅ **성능 최적화**: 최대한 빠른 처리
- ❌ **순서 보장 안 됨**: 메시지 순서가 섞일 수 있음

### 6.3 Partition Key를 활용한 순서 보장

**같은 Key는 같은 파티션으로, 다른 Key는 병렬 처리:**

```java
// Producer 측: 같은 OrderId는 같은 파티션으로
kafkaTemplate.send("order-events", orderId.toString(), event);

// Consumer 측: 파티션별로 순서 보장, 파티션 간은 병렬
@KafkaListener(topics = "order-events", concurrency = "3")
public void handleOrderEvent(OrderEvent event) {
    // 같은 파티션 내에서는 순서 보장
    // 다른 파티션은 병렬 처리
    orderService.processOrderEvent(event);
}
```

**특징:**
- ✅ **부분적 순서 보장**: 같은 Partition Key 내에서만 순서 보장
- ✅ **병렬 처리**: 다른 Key는 병렬 처리 가능
- ✅ **성능과 순서의 균형**: 최적의 Trade-off

---

## 7. 실제 사용 시나리오별 권장 사항

### 7.1 단순 메시지 처리 (순서 불필요)

**권장: @KafkaListener + 높은 Concurrency**

```java
@Component
@Slf4j
public class SimpleEventConsumer {
    
    @KafkaListener(topics = "notifications", concurrency = "10")
    public void handleNotification(NotificationEvent event) {
        // 순서가 중요하지 않은 알림 처리
        notificationService.sendNotification(event);
    }
}
```

**설정:**
- Concurrency: 파티션 수보다 높게 설정 (예: 10)
- 배치 처리: 활성화하여 성능 최적화

### 7.2 순서가 중요한 메시지 처리

**권장: @KafkaListener + Concurrency=1 (또는 Partition Key 활용)**

```java
@Component
@Slf4j
public class OrderedEventConsumer {
    
    // Concurrency=1로 순서 보장
    @KafkaListener(topics = "order-state-changes", concurrency = "1")
    public void handleOrderStateChange(OrderStateChangeEvent event) {
        // 순서가 보장되어 처리
        orderService.updateOrderState(event);
    }
}
```

**또는 Partition Key 활용:**

```java
// Producer: 같은 주문 ID는 같은 파티션으로
kafkaTemplate.send("order-state-changes", orderId.toString(), event);

// Consumer: 파티션별 순서 보장, 파티션 간 병렬
@KafkaListener(topics = "order-state-changes", concurrency = "3")
public void handleOrderStateChange(OrderStateChangeEvent event) {
    // 같은 주문 ID는 순서 보장, 다른 주문은 병렬 처리
    orderService.updateOrderState(event);
}
```

### 7.3 높은 처리량이 필요한 경우

**권장: @KafkaListener + 배치 처리 + 높은 Concurrency**

```java
@Component
@Slf4j
public class HighThroughputConsumer {
    
    @KafkaListener(topics = "logs", 
                   containerFactory = "batchKafkaListenerContainerFactory",
                   concurrency = "10")
    public void handleLogsBatch(
            List<ConsumerRecord<String, LogEvent>> records) {
        
        // 배치 처리로 높은 처리량
        logService.processLogsBatch(
            records.stream()
                .map(ConsumerRecord::value)
                .collect(Collectors.toList())
        );
    }
}
```

**설정:**
- 배치 처리: 활성화
- `max-poll-records`: 500 이상
- Concurrency: 파티션 수보다 높게

### 7.4 수동 제어가 필요한 경우

**권장: 수동 Polling (성능보다 제어가 중요한 경우)**

```java
@Service
@Slf4j
public class ManualControlConsumer {
    
    @Autowired
    private KafkaConsumer<String, Object> kafkaConsumer;
    
    public void consumeWithManualControl() {
        kafkaConsumer.subscribe(Collections.singletonList("important-topic"));
        
        while (true) {
            ConsumerRecords<String, Object> records = 
                kafkaConsumer.poll(Duration.ofMillis(100));
            
            for (ConsumerRecord<String, Object> record : records) {
                // 복잡한 처리 로직
                if (shouldProcess(record)) {
                    processMessage(record.value());
                    kafkaConsumer.commitSync();
                } else {
                    // 특정 조건에서 Skip
                    skipMessage(record);
                }
            }
        }
    }
}
```

**사용 사례:**
- 복잡한 조건부 처리
- 특정 메시지 Skip
- 완전한 제어가 필요한 경우

---

## 8. Best Practice

### 8.1 DO (해야 할 것)

1. **@KafkaListener 기본 사용**
   ```java
   // ✅ 대부분의 경우 @KafkaListener 사용
   @KafkaListener(topics = "topic-name")
   public void handle(Event event) { }
   ```

2. **Concurrency 최적화**
   ```java
   // ✅ 파티션 수와 일치하거나 약간 높게 설정
   @KafkaListener(topics = "topic", concurrency = "3")  // 파티션 3개
   ```

3. **배치 처리 활용**
   ```yaml
   # ✅ 높은 처리량이 필요한 경우
   spring:
     kafka:
       consumer:
         max-poll-records: 500
       listener:
         type: batch
   ```

4. **Partition Key 활용**
   ```java
   // ✅ 순서가 중요한 경우 Partition Key 사용
   kafkaTemplate.send("topic", key, value);
   ```

### 8.2 DON'T (하지 말아야 할 것)

1. **불필요한 수동 Polling 금지**
   ```java
   // ❌ 대부분의 경우 불필요
   while (true) {
       consumer.poll(Duration.ofMillis(100));
   }
   ```

2. **Concurrency 과다 설정 금지**
   ```java
   // ❌ 파티션보다 훨씬 많이 설정하면 유휴 스레드 발생
   @KafkaListener(topics = "topic", concurrency = "100")  // 파티션 3개인 경우
   ```

3. **순서 보장이 필요한데 Concurrency 높게 설정 금지**
   ```java
   // ❌ 순서가 중요한데 Concurrency를 높게 설정
   @KafkaListener(topics = "ordered-topic", concurrency = "10")
   ```

---

## 9. 실제 측정 예시

### 9.1 성능 테스트 코드

```java
@SpringBootTest
@Slf4j
public class KafkaConsumerPerformanceTest {
    
    @Autowired
    private OrderService orderService;
    
    @Test
    public void testKafkaListenerPerformance() {
        // 메시지 전송
        int messageCount = 10000;
        long startTime = System.currentTimeMillis();
        
        for (int i = 0; i < messageCount; i++) {
            OrderCreatedEvent event = new OrderCreatedEvent("order-" + i);
            // 메시지 전송 (Producer)
            kafkaTemplate.send("order-created", event.getOrderId(), event);
        }
        
        // Consumer 처리 대기
        Thread.sleep(5000);  // 처리 시간 대기
        
        long duration = System.currentTimeMillis() - startTime;
        log.info("KafkaListener performance: {} messages in {} ms ({} msg/sec)", 
            messageCount, duration, messageCount * 1000.0 / duration);
    }
}
```

### 9.2 예상 결과

**@KafkaListener (concurrency=5, 배치 처리):**
- 10,000건: 약 2초
- 처리량: 약 5,000 msg/sec

**@KafkaListener (concurrency=1, 단일 처리):**
- 10,000건: 약 100초
- 처리량: 약 100 msg/sec

**수동 Polling (동기):**
- 10,000건: 약 100초
- 처리량: 약 100 msg/sec

**배치 처리 + 높은 Concurrency가 가장 빠름**

---

## 마무리

**핵심 포인트:**

1. **@KafkaListener는 기본적으로 비동기**: 메시지 수신과 처리가 분리
2. **수동 Polling은 동기적**: 순차 처리로 성능 제한
3. **Concurrency로 병렬 처리**: 파티션 수와 일치하게 설정
4. **배치 처리로 성능 향상**: 여러 메시지를 한 번에 처리
5. **순서 보장은 Trade-off**: 성능을 포기하거나 Partition Key 활용

**권장 사항:**

- **대부분의 경우**: @KafkaListener 사용 (비동기)
- **높은 처리량 필요**: 배치 처리 + 높은 Concurrency
- **순서 보장 필요**: Concurrency=1 또는 Partition Key 활용
- **완전한 제어 필요**: 수동 Polling (성능 포기)

**Spring Kafka 3.3.11의 기본 설정:**
- `@KafkaListener`는 비동기로 동작
- Concurrency 기본값: 1 (단일 스레드)
- 배치 처리: 비활성화 (단일 메시지 처리)

**성능 최적화가 필요한 경우:**
- Concurrency를 파티션 수와 일치하게 설정
- 배치 처리 활성화 (`max-poll-records`, `listener.type: batch`)
- 불필요한 순서 보장 제거

Spring Kafka의 `@KafkaListener`는 기본적으로 비동기 방식으로 동작하므로, 대부분의 경우 추가 설정 없이도 충분한 성능을 얻을 수 있습니다. 하지만 **높은 처리량이 필요한 경우**에는 Concurrency와 배치 처리 설정을 통해 성능을 최적화해야 합니다. 🚀

다음 글에서는 **Pinpoint APM으로 Spring Boot 3.x / Java 21 애플리케이션을 모니터링하는 방법**을 정리해보겠습니다.
