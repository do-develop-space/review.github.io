---
layout: post
title: "Kafka Dead Letter Queue (DLQ) 완전 가이드: 실패 메시지 관리와 재처리 전략"
date: 2026-01-17
categories: [kafka, architecture, spring]
tags: [Kafka, DLQ, DeadLetterQueue, 실패메시지, 재처리, 모니터링, SpringKafka, 운영전략]
---

이전 글에서 Kafka `acks` 옵션을 통해 메시지의 내구성과 안정성을 보장하는 방법을 다뤘습니다. 하지만 메시지가 브로커에 안전하게 저장되어도, **Consumer에서 처리 실패**가 발생할 수 있습니다. 재시도 후에도 실패한 메시지를 어떻게 관리하고 재처리할지가 중요한 문제입니다.

이번 글에서는 **Dead Letter Queue (DLQ)**를 통해 실패한 메시지를 효과적으로 관리하고 재처리하는 전략을 정리해보겠습니다.

---

## 1. DLQ란 무엇인가?

### 1.1 DLQ의 개념

**Dead Letter Queue (DLQ, 데드 레터 큐):**

- **최대 재시도 횟수를 초과한 메시지**를 저장하는 특별한 토픽
- **수동 개입이 필요한 메시지**를 별도로 관리
- 운영자가 문제를 분석하고 수동으로 재처리 가능
- **메시지 손실 방지** 및 **문제 추적** 용이

**DLQ의 역할:**

```
정상 메시지 처리
    ↓
처리 실패 발생
    ↓
재시도 (최대 N번)
    ↓
재시도 실패
    ↓
DLQ로 이동 ← 여기서 관리
    ↓
운영자 분석 및 재처리
```

### 1.2 DLQ가 필요한 이유

**문제 상황:**

1. **재시도 실패 후 메시지 손실**
   - 재시도 횟수 초과 후 메시지가 버려짐
   - 중요한 비즈니스 로직 누락 가능

2. **실패 원인 추적 어려움**
   - 실패한 메시지가 사라져 원인 분석 불가
   - 동일한 문제가 반복 발생

3. **수동 재처리 불가능**
   - 문제 해결 후에도 메시지를 재처리할 수 없음
   - 데이터 일관성 문제 발생

**DLQ의 해결책:**

- ✅ 실패한 메시지 보존
- ✅ 실패 원인 메타데이터 저장
- ✅ 수동 재처리 가능
- ✅ 모니터링 및 알림 연동

---

## 2. DLQ 설계 전략

### 2.1 토픽 구조 설계

**DLQ 토픽 네이밍 전략:**

```
원본 토픽: order-created
DLQ 토픽: order-created-dlt  (Dead Letter Topic)

또는

원본 토픽: order-created
DLQ 토픽: dlq.order-created
```

**권장 사항:**

- 원본 토픽 이름을 포함하여 추적 용이
- `-dlt` 또는 `dlq.` 접두사/접미사 사용
- 일관된 네이밍 규칙 적용

**토픽 설정:**

```yaml
# DLQ 토픽 설정 예시
topic: order-created-dlt
partitions: 3  # 원본 토픽과 동일하게 설정 권장
replication-factor: 3
retention-ms: 604800000  # 7일 (실패 메시지는 오래 보관)
cleanup-policy: delete
```

### 2.2 DLQ 메시지 형식 설계

**DLQ 메시지에 포함해야 할 정보:**

```java
@Data
@Builder
public class DLQMessage {
    // 원본 메시지 정보
    private String originalTopic;      // 원본 토픽
    private String originalPartition;  // 원본 파티션
    private Long originalOffset;       // 원본 오프셋
    private String originalKey;        // 원본 키
    private Object originalPayload;    // 원본 메시지 본문
    
    // 실패 정보
    private String errorMessage;      // 에러 메시지
    private String errorStacktrace;    // 스택 트레이스
    private String errorType;          // 예외 타입
    private LocalDateTime failedAt;     // 실패 시각
    
    // 재시도 정보
    private Integer retryCount;        // 재시도 횟수
    private List<LocalDateTime> retryTimestamps;  // 재시도 시각들
    
    // 컨텍스트 정보
    private String consumerGroup;      // 컨슈머 그룹
    private String applicationName;    // 애플리케이션 이름
    private Map<String, String> metadata;  // 추가 메타데이터
}
```

**장점:**

- 원본 메시지 복원 가능
- 실패 원인 추적 용이
- 재시도 이력 확인 가능
- 디버깅 및 분석 용이

---

## 3. DLQ 구현 방법

### 3.1 Spring Kafka @RetryableTopic 활용

**Spring Kafka 3.3.11에서 제공하는 자동 DLQ 처리:**

Spring Kafka 3.3.11에서는 `@RetryableTopic` 어노테이션을 사용하여 재시도 및 DLQ 처리를 자동화할 수 있습니다.

**의존성 설정 (build.gradle):**

```gradle
dependencies {
    implementation 'org.springframework.kafka:spring-kafka:3.3.11'
}
```

**Configuration 설정:**

```java
@Configuration
@EnableKafka
public class KafkaConfig {
    
    @Bean
    public KafkaTemplate<String, Object> kafkaTemplate(
            ProducerFactory<String, Object> producerFactory) {
        return new KafkaTemplate<>(producerFactory);
    }
    
    @Bean
    public ConsumerFactory<String, Object> consumerFactory() {
        Map<String, Object> props = new HashMap<>();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "order-consumer-group");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, JsonDeserializer.class);
        props.put(JsonDeserializer.TRUSTED_PACKAGES, "*");
        return new DefaultKafkaConsumerFactory<>(props);
    }
    
    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, Object> 
        kafkaListenerContainerFactory() {
        ConcurrentKafkaListenerContainerFactory<String, Object> factory = 
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory());
        factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.MANUAL_IMMEDIATE);
        return factory;
    }
}
```

**Consumer에서 @RetryableTopic 사용:**

```java
@Component
@Slf4j
public class OrderConsumer {
    
    @Autowired
    private OrderService orderService;
    
    @Autowired
    private DLQService dlqService;
    
    @Autowired
    private AlertService alertService;
    
    @RetryableTopic(
        attempts = "4",  // 최대 4번 재시도
        backoff = @Backoff(delay = 1000, multiplier = 2, maxDelay = 10000),  // 1초 → 2초 → 4초 → 8초 → 최대 10초
        dltStrategy = DltStrategy.FAIL_ON_ERROR,  // 최종 실패 시 DLQ로 전송
        include = {RuntimeException.class},  // 재시도할 예외
        exclude = {IllegalArgumentException.class},  // 재시도하지 않을 예외
        autoCreateTopics = "true",  // 자동으로 재시도 토픽 및 DLQ 토픽 생성
        topicSuffixingStrategy = TopicSuffixingStrategy.SUFFIX_WITH_INDEX_VALUE  // 토픽 이름에 인덱스 추가
    )
    @KafkaListener(topics = "order-created")
    public void consumeOrderCreated(OrderCreatedEvent event) {
        log.info("Processing order: {}", event.getOrderId());
        
        // 비즈니스 로직 처리
        orderService.processOrder(event);
        
        // 예외 발생 시 자동으로 재시도 토픽으로 이동
        // 재시도 실패 시 DLQ로 이동
    }
    
    // DLQ 메시지 핸들러
    @DltHandler
    public void handleDLQ(
            @Payload OrderCreatedEvent event,
            @Header(KafkaHeaders.RECEIVED_TOPIC) String originalTopic,
            @Header(KafkaHeaders.RECEIVED_PARTITION_ID) int partition,
            @Header(KafkaHeaders.OFFSET) long offset) {
        
        log.error("DLQ message received: topic={}, partition={}, offset={}, event={}", 
            originalTopic, partition, offset, event);
        
        // DLQ 메시지 저장 (원본 토픽 정보 포함)
        DLQMessage dlqMessage = DLQMessage.builder()
            .originalTopic(originalTopic)
            .originalPartition(String.valueOf(partition))
            .originalOffset(offset)
            .originalKey(event.getOrderId().toString())
            .originalPayload(event)
            .failedAt(LocalDateTime.now())
            .build();
        
        dlqService.saveDLQMessage(dlqMessage);
        
        // 알림 발송
        alertService.sendAlert("DLQ message received", event);
    }
}
```

**자동 생성되는 토픽:**

```
order-created              (원본)
order-created-retry-0      (1차 재시도)
order-created-retry-1      (2차 재시도)
order-created-retry-2      (3차 재시도)
order-created-dlt          (DLQ)
```

**필요한 Import 문:**

```java
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.annotation.RetryableTopic;
import org.springframework.kafka.annotation.DltHandler;
import org.springframework.kafka.annotation.Backoff;
import org.springframework.kafka.retrytopic.DltStrategy;
import org.springframework.kafka.retrytopic.TopicSuffixingStrategy;
import org.springframework.kafka.support.KafkaHeaders;
import org.springframework.kafka.support.Acknowledgment;
import org.springframework.messaging.handler.annotation.Header;
import org.springframework.messaging.handler.annotation.Payload;
```

**application.yml 설정 (Spring Kafka 3.3.11):**

```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    
    # Producer 설정
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      acks: all  # 메시지 내구성 보장
    
    # Consumer 설정
    consumer:
      group-id: order-consumer-group
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      properties:
        spring.json.trusted.packages: "*"  # 모든 패키지에서 역직렬화 허용
      enable-auto-commit: false  # 수동 커밋 사용
      auto-offset-reset: earliest
    
    # Admin 설정 (토픽 자동 생성용)
    admin:
      properties:
        bootstrap.servers: localhost:9092
```

**장점:**

- ✅ 자동으로 재시도 토픽 및 DLQ 토픽 생성
- ✅ 지연 발행(backoff) 자동 처리
- ✅ 설정이 간단하고 명확
- ✅ Spring Kafka 3.3.11에서 안정적으로 지원

**단점:**

- ❌ 세밀한 제어가 어려울 수 있음
- ❌ Spring Kafka 2.7+ 버전 필요 (현재 프로젝트는 3.3.11 사용)

### 3.2 수동 DLQ 구현

**더 세밀한 제어가 필요한 경우 (수동 DLQ 구현):**

Spring Kafka 3.3.11에서 `@RetryableTopic`을 사용하지 않고 수동으로 DLQ를 구현하는 방법:

```java
@Component
@Slf4j
public class OrderConsumer {
    
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;
    
    @Autowired
    private DLQService dlqService;
    
    @KafkaListener(topics = "order-created", 
                   containerFactory = "kafkaListenerContainerFactory")
    public void consumeOrderCreated(
            @Payload OrderCreatedEvent event,
            @Header(KafkaHeaders.RECEIVED_TOPIC) String topic,
            @Header(KafkaHeaders.RECEIVED_PARTITION_ID) int partition,
            @Header(KafkaHeaders.OFFSET) long offset,
            Acknowledgment acknowledgment) {
        
        int maxRetries = 3;
        int currentRetry = getRetryCount(event);  // 메시지 헤더에서 재시도 횟수 확인
        
        try {
            // 비즈니스 로직 처리
            orderService.processOrder(event);
            
            // 성공 시 offset 커밋
            acknowledgment.acknowledge();
            
        } catch (RetryableException e) {
            // 재시도 가능한 예외
            if (currentRetry < maxRetries) {
                // 재시도 토픽으로 전송
                sendToRetryTopic(topic, event, currentRetry + 1, e);
                acknowledgment.acknowledge();
            } else {
                // 최대 재시도 횟수 초과 → DLQ로 전송
                sendToDLQ(topic, partition, offset, event, e, currentRetry);
                acknowledgment.acknowledge();
            }
            
        } catch (NonRetryableException e) {
            // 재시도 불가능한 예외 (비즈니스 로직 오류 등)
            // 즉시 DLQ로 전송
            sendToDLQ(topic, partition, offset, event, e, currentRetry);
            acknowledgment.acknowledge();
        }
    }
    
    private void sendToDLQ(
            String originalTopic,
            int partition,
            long offset,
            OrderCreatedEvent event,
            Exception error,
            int retryCount) {
        
        // DLQ 메시지 생성
        DLQMessage dlqMessage = DLQMessage.builder()
            .originalTopic(originalTopic)
            .originalPartition(String.valueOf(partition))
            .originalOffset(offset)
            .originalKey(event.getOrderId().toString())
            .originalPayload(event)
            .errorMessage(error.getMessage())
            .errorStacktrace(getStackTrace(error))
            .errorType(error.getClass().getName())
            .failedAt(LocalDateTime.now())
            .retryCount(retryCount)
            .consumerGroup("order-consumer-group")
            .applicationName("order-service")
            .build();
        
        // DLQ 토픽으로 전송
        kafkaTemplate.send("order-created-dlt", event.getOrderId().toString(), dlqMessage);
        
        // DB에도 저장 (조회 및 분석 용이)
        dlqService.saveDLQMessage(dlqMessage);
        
        log.error("Message sent to DLQ: topic={}, partition={}, offset={}, error={}",
            originalTopic, partition, offset, error.getMessage());
    }
    
    private void sendToRetryTopic(
            String originalTopic,
            OrderCreatedEvent event,
            int retryCount,
            Exception error) {
        
        // 재시도 토픽으로 전송 (지연 발행)
        RetryMessage retryMessage = RetryMessage.builder()
            .originalEvent(event)
            .retryCount(retryCount)
            .lastError(error.getMessage())
            .build();
        
        // 지연 시간 계산 (exponential backoff)
        long delayMs = (long) Math.pow(2, retryCount) * 1000;
        
        // 지연 발행 (Kafka의 delayed message는 별도 구현 필요)
        kafkaTemplate.send("order-created-retry", event.getOrderId().toString(), retryMessage);
    }
    
    private String getStackTrace(Exception e) {
        StringWriter sw = new StringWriter();
        PrintWriter pw = new PrintWriter(sw);
        e.printStackTrace(pw);
        return sw.toString();
    }
}
```

### 3.3 DLQ 서비스 구현

**DLQ 메시지 저장 및 관리:**

```java
@Service
@Slf4j
public class DLQService {
    
    @Autowired
    private DLQRepository dlqRepository;
    
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;
    
    public void saveDLQMessage(DLQMessage message) {
        // DB에 저장
        dlqRepository.save(message);
        
        log.info("DLQ message saved: id={}, topic={}, error={}",
            message.getId(), message.getOriginalTopic(), message.getErrorMessage());
    }
    
    // DLQ 메시지 조회
    public Page<DLQMessage> getDLQMessages(
            String topic,
            LocalDateTime from,
            LocalDateTime to,
            Pageable pageable) {
        
        return dlqRepository.findByOriginalTopicAndFailedAtBetween(
            topic, from, to, pageable
        );
    }
    
    // DLQ 메시지 재처리
    @Transactional
    public void reprocessDLQMessage(Long dlqMessageId) {
        DLQMessage dlqMessage = dlqRepository.findById(dlqMessageId)
            .orElseThrow(() -> new IllegalArgumentException("DLQ message not found"));
        
        try {
            // 원본 토픽으로 재전송
            kafkaTemplate.send(
                dlqMessage.getOriginalTopic(),
                dlqMessage.getOriginalKey(),
                dlqMessage.getOriginalPayload()
            );
            
            // DLQ에서 제거 (또는 상태 변경)
            dlqMessage.setStatus(DLQStatus.REPROCESSED);
            dlqRepository.save(dlqMessage);
            
            log.info("DLQ message {} reprocessed successfully", dlqMessageId);
            
        } catch (Exception e) {
            log.error("Failed to reprocess DLQ message: {}", dlqMessageId, e);
            throw e;
        }
    }
    
    // DLQ 메시지 일괄 재처리
    @Transactional
    public void reprocessDLQMessages(List<Long> dlqMessageIds) {
        for (Long id : dlqMessageIds) {
            try {
                reprocessDLQMessage(id);
            } catch (Exception e) {
                log.error("Failed to reprocess DLQ message: {}", id, e);
                // 계속 진행 (일부 실패해도 나머지 처리)
            }
        }
    }
    
    // DLQ 메시지 삭제
    @Transactional
    public void deleteDLQMessage(Long dlqMessageId) {
        dlqRepository.deleteById(dlqMessageId);
        log.info("DLQ message {} deleted", dlqMessageId);
    }
}
```

**DLQ 엔티티:**

```java
@Entity
@Table(name = "dlq_messages")
@Data
public class DLQMessage {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false)
    private String originalTopic;
    
    @Column(nullable = false)
    private String originalPartition;
    
    @Column(nullable = false)
    private Long originalOffset;
    
    @Column(nullable = false)
    private String originalKey;
    
    @Column(columnDefinition = "TEXT")
    private String originalPayload;  // JSON 문자열로 저장
    
    @Column(columnDefinition = "TEXT")
    private String errorMessage;
    
    @Column(columnDefinition = "TEXT")
    private String errorStacktrace;
    
    @Column(nullable = false)
    private String errorType;
    
    @Column(nullable = false)
    private LocalDateTime failedAt;
    
    @Column(nullable = false)
    private Integer retryCount;
    
    @Enumerated(EnumType.STRING)
    private DLQStatus status = DLQStatus.PENDING;
    
    private LocalDateTime reprocessedAt;
    
    @Column(columnDefinition = "TEXT")
    private String metadata;  // JSON 문자열로 저장
}

public enum DLQStatus {
    PENDING,      // 대기 중
    REPROCESSED,  // 재처리됨
    DELETED,      // 삭제됨
    IGNORED       // 무시됨
}
```

### 3.4 Producer DLQ 구현

**Producer DLQ의 필요성:**

Consumer DLQ와 달리, **Producer에서는 Spring Kafka가 자동으로 DLQ를 제공하지 않습니다**. Producer에서 메시지 전송이 실패할 경우, 수동으로 DLQ 처리를 구현해야 합니다.

**Producer 전송 실패 시나리오:**

1. **네트워크 오류**: 브로커와 연결 실패
2. **브로커 장애**: 브로커가 응답하지 않음
3. **토픽 미존재**: 전송하려는 토픽이 없음
4. **권한 오류**: Producer에 토픽 쓰기 권한 없음
5. **Serialization 오류**: 메시지 직렬화 실패
6. **재시도 실패**: 최대 재시도 횟수 초과

**Producer DLQ 구현 예시:**

```java
@Service
@Slf4j
public class OrderEventProducer {
    
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;
    
    @Autowired
    private ProducerDLQService producerDLQService;
    
    private static final String TOPIC = "order-created";
    private static final String PRODUCER_DLQ_TOPIC = "producer-dlq.order-created";
    
    /**
     * 메시지 전송 (Producer DLQ 포함)
     */
    public void sendOrderEvent(OrderCreatedEvent event) {
        String partitionKey = event.getOrderId().toString();
        
        try {
            // 메시지 전송
            ListenableFuture<SendResult<String, Object>> future = 
                kafkaTemplate.send(TOPIC, partitionKey, event);
            
            // 비동기 Callback 처리
            future.addCallback(
                result -> {
                    // 성공
                    log.info("Message sent successfully: topic={}, partition={}, offset={}, key={}",
                        result.getRecordMetadata().topic(),
                        result.getRecordMetadata().partition(),
                        result.getRecordMetadata().offset(),
                        partitionKey);
                },
                failure -> {
                    // 실패 → Producer DLQ로 전송
                    log.error("Failed to send message: topic={}, key={}, error={}",
                        TOPIC, partitionKey, failure.getMessage());
                    
                    sendToProducerDLQ(event, partitionKey, failure);
                }
            );
            
        } catch (Exception e) {
            // 동기 전송 시 예외 발생
            log.error("Exception while sending message: topic={}, key={}",
                TOPIC, partitionKey, e);
            
            sendToProducerDLQ(event, partitionKey, e);
        }
    }
    
    /**
     * Producer DLQ로 메시지 전송
     */
    private void sendToProducerDLQ(
            OrderCreatedEvent event,
            String partitionKey,
            Throwable failure) {
        
        try {
            // Producer DLQ 메시지 생성
            ProducerDLQMessage dlqMessage = ProducerDLQMessage.builder()
                .originalTopic(TOPIC)
                .originalKey(partitionKey)
                .originalPayload(event)
                .errorMessage(failure.getMessage())
                .errorStacktrace(getStackTrace(failure))
                .errorType(failure.getClass().getName())
                .failedAt(LocalDateTime.now())
                .build();
            
            // Producer DLQ 토픽으로 전송
            kafkaTemplate.send(PRODUCER_DLQ_TOPIC, partitionKey, dlqMessage)
                .addCallback(
                    result -> {
                        log.info("Message sent to Producer DLQ: topic={}, key={}",
                            PRODUCER_DLQ_TOPIC, partitionKey);
                        
                        // DB에도 저장 (추적 및 분석 용이)
                        producerDLQService.saveProducerDLQMessage(dlqMessage);
                    },
                    dlqFailure -> {
                        // Producer DLQ 전송도 실패한 경우
                        log.error("Failed to send to Producer DLQ: key={}", 
                            partitionKey, dlqFailure);
                        
                        // 최후의 수단: DB에만 저장
                        producerDLQService.saveProducerDLQMessage(dlqMessage);
                    }
                );
                
        } catch (Exception e) {
            log.error("Exception while sending to Producer DLQ: key={}", 
                partitionKey, e);
            
            // DB에만 저장
            ProducerDLQMessage dlqMessage = ProducerDLQMessage.builder()
                .originalTopic(TOPIC)
                .originalKey(partitionKey)
                .originalPayload(event)
                .errorMessage(failure.getMessage())
                .errorStacktrace(getStackTrace(failure))
                .errorType(failure.getClass().getName())
                .failedAt(LocalDateTime.now())
                .build();
            
            producerDLQService.saveProducerDLQMessage(dlqMessage);
        }
    }
    
    private String getStackTrace(Throwable throwable) {
        StringWriter sw = new StringWriter();
        PrintWriter pw = new PrintWriter(sw);
        throwable.printStackTrace(pw);
        return sw.toString();
    }
}
```

**재시도 후 실패 시 Producer DLQ 전송:**

```java
@Service
@Slf4j
public class OrderEventProducerWithRetry {
    
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;
    
    @Autowired
    private ProducerDLQService producerDLQService;
    
    private static final String TOPIC = "order-created";
    private static final String PRODUCER_DLQ_TOPIC = "producer-dlq.order-created";
    private static final int MAX_RETRIES = 3;
    
    /**
     * 재시도 로직이 포함된 메시지 전송
     */
    public void sendOrderEventWithRetry(OrderCreatedEvent event) {
        String partitionKey = event.getOrderId().toString();
        sendWithRetry(event, partitionKey, 0);
    }
    
    private void sendWithRetry(
            OrderCreatedEvent event,
            String partitionKey,
            int retryCount) {
        
        try {
            ListenableFuture<SendResult<String, Object>> future = 
                kafkaTemplate.send(TOPIC, partitionKey, event);
            
            future.addCallback(
                result -> {
                    log.info("Message sent successfully: topic={}, partition={}, offset={}, key={}, retryCount={}",
                        result.getRecordMetadata().topic(),
                        result.getRecordMetadata().partition(),
                        result.getRecordMetadata().offset(),
                        partitionKey,
                        retryCount);
                },
                failure -> {
                    log.warn("Failed to send message (retry {}/{}): topic={}, key={}, error={}",
                        retryCount + 1, MAX_RETRIES, TOPIC, partitionKey, failure.getMessage());
                    
                    // 재시도 가능 여부 확인
                    if (retryCount < MAX_RETRIES && isRetryableException(failure)) {
                        // 재시도
                        scheduleRetry(event, partitionKey, retryCount + 1);
                    } else {
                        // 최대 재시도 횟수 초과 또는 재시도 불가능한 오류 → Producer DLQ로 전송
                        log.error("Max retries exceeded or non-retryable error: topic={}, key={}, retryCount={}",
                            TOPIC, partitionKey, retryCount);
                        
                        sendToProducerDLQ(event, partitionKey, failure, retryCount);
                    }
                }
            );
            
        } catch (Exception e) {
            log.error("Exception while sending message: topic={}, key={}, retryCount={}",
                TOPIC, partitionKey, retryCount, e);
            
            if (retryCount < MAX_RETRIES && isRetryableException(e)) {
                scheduleRetry(event, partitionKey, retryCount + 1);
            } else {
                sendToProducerDLQ(event, partitionKey, e, retryCount);
            }
        }
    }
    
    /**
     * 재시도 가능한 예외인지 확인
     */
    private boolean isRetryableException(Throwable throwable) {
        // 재시도 가능한 예외: 네트워크 오류, 브로커 장애 등
        return throwable instanceof KafkaException ||
               throwable instanceof TimeoutException ||
               throwable instanceof NetworkException ||
               (throwable.getCause() != null && isRetryableException(throwable.getCause()));
    }
    
    /**
     * 지수 백오프로 재시도 스케줄링
     */
    private void scheduleRetry(
            OrderCreatedEvent event,
            String partitionKey,
            int retryCount) {
        
        // 지수 백오프: 1초 → 2초 → 4초
        long delayMs = (long) Math.pow(2, retryCount) * 1000;
        
        log.info("Scheduling retry {}: key={}, delay={}ms", 
            retryCount, partitionKey, delayMs);
        
        // 스케줄러를 사용하여 지연 재시도
        // (실제 구현에서는 @Async, ScheduledExecutorService, 또는 RabbitMQ Delayed Message 등 사용)
        CompletableFuture.delayedExecutor(delayMs, TimeUnit.MILLISECONDS)
            .execute(() -> sendWithRetry(event, partitionKey, retryCount));
    }
    
    /**
     * Producer DLQ로 메시지 전송
     */
    private void sendToProducerDLQ(
            OrderCreatedEvent event,
            String partitionKey,
            Throwable failure,
            int retryCount) {
        
        ProducerDLQMessage dlqMessage = ProducerDLQMessage.builder()
            .originalTopic(TOPIC)
            .originalKey(partitionKey)
            .originalPayload(event)
            .errorMessage(failure.getMessage())
            .errorStacktrace(getStackTrace(failure))
            .errorType(failure.getClass().getName())
            .failedAt(LocalDateTime.now())
            .retryCount(retryCount)
            .build();
        
        try {
            kafkaTemplate.send(PRODUCER_DLQ_TOPIC, partitionKey, dlqMessage)
                .addCallback(
                    result -> {
                        log.info("Message sent to Producer DLQ: topic={}, key={}, retryCount={}",
                            PRODUCER_DLQ_TOPIC, partitionKey, retryCount);
                        producerDLQService.saveProducerDLQMessage(dlqMessage);
                    },
                    dlqFailure -> {
                        log.error("Failed to send to Producer DLQ: key={}", 
                            partitionKey, dlqFailure);
                        producerDLQService.saveProducerDLQMessage(dlqMessage);
                    }
                );
        } catch (Exception e) {
            log.error("Exception while sending to Producer DLQ: key={}", 
                partitionKey, e);
            producerDLQService.saveProducerDLQMessage(dlqMessage);
        }
    }
    
    private String getStackTrace(Throwable throwable) {
        StringWriter sw = new StringWriter();
        PrintWriter pw = new PrintWriter(sw);
        throwable.printStackTrace(pw);
        return sw.toString();
    }
}
```

**Producer DLQ 메시지 구조:**

```java
@Data
@Builder
public class ProducerDLQMessage {
    // 원본 메시지 정보
    private String originalTopic;      // 원본 토픽
    private String originalKey;        // 원본 키
    private Object originalPayload;    // 원본 메시지 본문
    
    // 실패 정보
    private String errorMessage;      // 에러 메시지
    private String errorStacktrace;    // 스택 트레이스
    private String errorType;          // 예외 타입
    private LocalDateTime failedAt;     // 실패 시각
    
    // 재시도 정보
    private Integer retryCount;        // 재시도 횟수
    
    // 컨텍스트 정보
    private String producerId;        // Producer 식별자
    private String applicationName;    // 애플리케이션 이름
    private Map<String, String> metadata;  // 추가 메타데이터
}
```

**Producer DLQ 서비스:**

```java
@Service
@Slf4j
public class ProducerDLQService {
    
    @Autowired
    private ProducerDLQRepository producerDLQRepository;
    
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;
    
    public void saveProducerDLQMessage(ProducerDLQMessage message) {
        // DB에 저장
        producerDLQRepository.save(message);
        
        log.info("Producer DLQ message saved: topic={}, key={}, error={}",
            message.getOriginalTopic(), message.getOriginalKey(), message.getErrorMessage());
    }
    
    /**
     * Producer DLQ 메시지 재처리 (원본 토픽으로 재전송)
     */
    @Transactional
    public void reprocessProducerDLQMessage(Long dlqMessageId) {
        ProducerDLQMessage dlqMessage = producerDLQRepository.findById(dlqMessageId)
            .orElseThrow(() -> new IllegalArgumentException("Producer DLQ message not found"));
        
        try {
            // 원본 토픽으로 재전송
            kafkaTemplate.send(
                dlqMessage.getOriginalTopic(),
                dlqMessage.getOriginalKey(),
                dlqMessage.getOriginalPayload()
            ).addCallback(
                result -> {
                    log.info("Producer DLQ message {} reprocessed successfully", dlqMessageId);
                    dlqMessage.setStatus(DLQStatus.REPROCESSED);
                    dlqMessage.setReprocessedAt(LocalDateTime.now());
                    producerDLQRepository.save(dlqMessage);
                },
                failure -> {
                    log.error("Failed to reprocess Producer DLQ message: {}", 
                        dlqMessageId, failure);
                    throw new RuntimeException("Failed to reprocess", failure);
                }
            );
            
        } catch (Exception e) {
            log.error("Failed to reprocess Producer DLQ message: {}", dlqMessageId, e);
            throw e;
        }
    }
    
    /**
     * Producer DLQ 메시지 조회
     */
    public Page<ProducerDLQMessage> getProducerDLQMessages(
            String topic,
            LocalDateTime from,
            LocalDateTime to,
            Pageable pageable) {
        
        return producerDLQRepository.findByOriginalTopicAndFailedAtBetween(
            topic, from, to, pageable
        );
    }
}
```

**Producer DLQ 설정 (application.yml):**

```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    
    # Producer 설정
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      acks: all  # 메시지 내구성 보장
      retries: 3  # 재시도 횟수
      retry-backoff-ms: 1000  # 재시도 간격
      max-in-flight-requests-per-connection: 5
      enable-idempotence: true  # 멱등성 보장
      properties:
        # 추가 Producer 설정
        linger.ms: 10
        batch.size: 16384
```

### 3.5 Consumer DLQ vs Producer DLQ 비교

**Consumer DLQ:**
- ✅ Spring Kafka `@RetryableTopic`으로 자동 처리
- ✅ Consumer에서 메시지 처리 실패 시 사용
- ✅ 재시도 토픽 자동 생성 및 관리
- ✅ `@DltHandler`로 DLQ 메시지 처리 가능

**Producer DLQ:**
- ❌ Spring Kafka가 자동으로 제공하지 않음
- ❌ Producer에서 메시지 전송 실패 시 사용
- ❌ 수동으로 Callback 처리 및 DLQ 전송 구현 필요
- ❌ Producer 재시도 설정은 있지만, 실패 후 DLQ는 수동 처리

**권장 사항:**

- **Consumer DLQ**: `@RetryableTopic` 사용 (자동화 권장)
- **Producer DLQ**: Callback 기반 수동 구현 (세밀한 제어 가능)
- **모니터링**: 둘 다 모니터링 및 알림 설정 필수

---

## 4. DLQ 모니터링 및 알림

### 4.1 DLQ 메시지 모니터링

**Consumer DLQ 메트릭 수집:**

```java
@Component
@Slf4j
public class DLQMetrics {
    
    @Autowired
    private MeterRegistry meterRegistry;
    
    @Autowired
    private DLQRepository dlqRepository;
    
    @Autowired
    private ProducerDLQRepository producerDLQRepository;
    
    // Consumer DLQ 메시지 수 카운트
    public void recordDLQMessage(String topic, String errorType) {
        meterRegistry.counter("kafka.dlq.consumer.count", 
            "topic", topic,
            "error_type", errorType
        ).increment();
    }
    
    // Producer DLQ 메시지 수 카운트
    public void recordProducerDLQMessage(String topic, String errorType) {
        meterRegistry.counter("kafka.dlq.producer.count", 
            "topic", topic,
            "error_type", errorType
        ).increment();
    }
    
    // DLQ 메시지 비율 (전체 메시지 대비)
    public void recordDLQRate(String topic, long totalMessages, long dlqMessages) {
        double rate = (double) dlqMessages / totalMessages;
        meterRegistry.gauge("kafka.dlq.rate", 
            Tags.of("topic", topic),
            rate
        );
    }
    
    // DLQ 메시지 재처리 성공률
    public void recordReprocessSuccess(String topic, boolean success) {
        meterRegistry.counter("kafka.dlq.reprocess",
            "topic", topic,
            "status", success ? "success" : "failure"
        ).increment();
    }
    
    // 주기적으로 Consumer DLQ 메시지 수 확인
    @Scheduled(fixedRate = 60000)  // 1분마다
    public void checkDLQMessages() {
        List<DLQMessage> pendingMessages = dlqRepository.findByStatus(DLQStatus.PENDING);
        
        // 토픽별로 그룹화
        Map<String, Long> countByTopic = pendingMessages.stream()
            .collect(Collectors.groupingBy(
                DLQMessage::getOriginalTopic,
                Collectors.counting()
            ));
        
        // 메트릭 기록
        countByTopic.forEach((topic, count) -> {
            meterRegistry.gauge("kafka.dlq.consumer.pending.count",
                Tags.of("topic", topic),
                count
            );
            
            // 임계값 초과 시 알림
            if (count > 10) {
                alertService.sendAlert(
                    String.format("Consumer DLQ 메시지가 %d개 쌓였습니다: %s", count, topic)
                );
            }
        });
    }
    
    // 주기적으로 Producer DLQ 메시지 수 확인
    @Scheduled(fixedRate = 60000)  // 1분마다
    public void checkProducerDLQMessages() {
        List<ProducerDLQMessage> pendingMessages = 
            producerDLQRepository.findByStatus(DLQStatus.PENDING);
        
        // 토픽별로 그룹화
        Map<String, Long> countByTopic = pendingMessages.stream()
            .collect(Collectors.groupingBy(
                ProducerDLQMessage::getOriginalTopic,
                Collectors.counting()
            ));
        
        // 메트릭 기록
        countByTopic.forEach((topic, count) -> {
            meterRegistry.gauge("kafka.dlq.producer.pending.count",
                Tags.of("topic", topic),
                count
            );
            
            // 임계값 초과 시 알림
            if (count > 10) {
                alertService.sendAlert(
                    String.format("Producer DLQ 메시지가 %d개 쌓였습니다: %s", count, topic)
                );
            }
        });
    }
}
```

### 4.2 알림 전략

**DLQ 메시지 발생 시 알림:**

```java
@Service
@Slf4j
public class DLQAlertService {
    
    @Autowired
    private SlackService slackService;
    
    @Autowired
    private EmailService emailService;
    
    public void sendDLQAlert(DLQMessage message) {
        // 즉시 알림 (Slack)
        slackService.sendMessage(
            "#kafka-alerts",
            String.format("""
                🚨 DLQ 메시지 발생
                
                토픽: %s
                파티션: %s
                오프셋: %d
                에러: %s
                재시도 횟수: %d
                발생 시각: %s
                
                [상세 보기](%s)
                """,
                message.getOriginalTopic(),
                message.getOriginalPartition(),
                message.getOriginalOffset(),
                message.getErrorMessage(),
                message.getRetryCount(),
                message.getFailedAt(),
                getDLQMessageUrl(message.getId())
            )
        );
        
        // 중요 토픽의 경우 이메일 알림
        if (isCriticalTopic(message.getOriginalTopic())) {
            emailService.sendEmail(
                "kafka-team@example.com",
                "DLQ 메시지 발생 알림",
                buildEmailContent(message)
            );
        }
    }
    
    // Consumer DLQ 메시지가 일정 수 이상 쌓이면 알림
    public void checkDLQThreshold(String topic, long count) {
        if (count > 10) {
            slackService.sendMessage(
                "#kafka-alerts",
                String.format("⚠️ Consumer DLQ 메시지가 %d개 쌓였습니다: %s", count, topic)
            );
        }
        
        if (count > 100) {
            // 긴급 알림
            slackService.sendMessage(
                "#kafka-urgent",
                String.format("🚨 긴급: Consumer DLQ 메시지가 %d개 쌓였습니다: %s", count, topic)
            );
        }
    }
    
    // Producer DLQ 메시지 발생 시 알림
    public void sendProducerDLQAlert(ProducerDLQMessage message) {
        // 즉시 알림 (Slack)
        slackService.sendMessage(
            "#kafka-alerts",
            String.format("""
                🚨 Producer DLQ 메시지 발생
                
                토픽: %s
                키: %s
                에러: %s
                재시도 횟수: %d
                발생 시각: %s
                
                [상세 보기](%s)
                """,
                message.getOriginalTopic(),
                message.getOriginalKey(),
                message.getErrorMessage(),
                message.getRetryCount(),
                message.getFailedAt(),
                getProducerDLQMessageUrl(message.getId())
            )
        );
        
        // 중요 토픽의 경우 이메일 알림
        if (isCriticalTopic(message.getOriginalTopic())) {
            emailService.sendEmail(
                "kafka-team@example.com",
                "Producer DLQ 메시지 발생 알림",
                buildProducerDLQEmailContent(message)
            );
        }
    }
    
    // Producer DLQ 메시지가 일정 수 이상 쌓이면 알림
    public void checkProducerDLQThreshold(String topic, long count) {
        if (count > 10) {
            slackService.sendMessage(
                "#kafka-alerts",
                String.format("⚠️ Producer DLQ 메시지가 %d개 쌓였습니다: %s", count, topic)
            );
        }
        
        if (count > 100) {
            // 긴급 알림
            slackService.sendMessage(
                "#kafka-urgent",
                String.format("🚨 긴급: Producer DLQ 메시지가 %d개 쌓였습니다: %s", count, topic)
            );
        }
    }
    
    private boolean isCriticalTopic(String topic) {
        return topic.contains("payment") || 
               topic.contains("order") || 
               topic.contains("inventory");
    }
}
```

### 4.3 대시보드 구성

**Grafana 대시보드 예시:**

```yaml
# DLQ 모니터링 대시보드
panels:
  - title: "DLQ 메시지 수 (토픽별)"
    query: "sum(kafka_dlq_count) by (topic)"
    type: "graph"
    
  - title: "DLQ 메시지 비율"
    query: "rate(kafka_dlq_rate[5m])"
    type: "graph"
    
  - title: "DLQ 재처리 성공률"
    query: "sum(kafka_dlq_reprocess{status='success'}) / sum(kafka_dlq_reprocess)"
    type: "stat"
    
  - title: "DLQ 대기 중인 메시지 수"
    query: "kafka_dlq_pending_count"
    type: "table"
```

---

## 5. DLQ 메시지 분석 및 재처리 전략

### 5.1 DLQ 메시지 분석

**에러 유형별 분류:**

```java
@Service
public class DLQAnalysisService {
    
    @Autowired
    private DLQRepository dlqRepository;
    
    public DLQAnalysisResult analyzeDLQMessages(
            String topic,
            LocalDateTime from,
            LocalDateTime to) {
        
        List<DLQMessage> messages = dlqRepository.findByOriginalTopicAndFailedAtBetween(
            topic, from, to
        );
        
        // 에러 유형별 그룹화
        Map<String, Long> errorTypeCount = messages.stream()
            .collect(Collectors.groupingBy(
                DLQMessage::getErrorType,
                Collectors.counting()
            ));
        
        // 에러 메시지 패턴 분석
        Map<String, Long> errorPatternCount = messages.stream()
            .collect(Collectors.groupingBy(
                msg -> extractErrorPattern(msg.getErrorMessage()),
                Collectors.counting()
            ));
        
        // 재시도 횟수 분포
        Map<Integer, Long> retryCountDistribution = messages.stream()
            .collect(Collectors.groupingBy(
                DLQMessage::getRetryCount,
                Collectors.counting()
            ));
        
        return DLQAnalysisResult.builder()
            .totalMessages(messages.size())
            .errorTypeCount(errorTypeCount)
            .errorPatternCount(errorPatternCount)
            .retryCountDistribution(retryCountDistribution)
            .build();
    }
    
    private String extractErrorPattern(String errorMessage) {
        // 에러 메시지에서 패턴 추출 (예: "NullPointerException", "Connection timeout" 등)
        if (errorMessage.contains("NullPointerException")) {
            return "NullPointerException";
        } else if (errorMessage.contains("timeout")) {
            return "Timeout";
        } else if (errorMessage.contains("connection")) {
            return "Connection Error";
        }
        return "Other";
    }
}
```

### 5.2 자동 재처리 전략

**조건부 자동 재처리:**

```java
@Service
@Slf4j
public class AutoReprocessService {
    
    @Autowired
    private DLQService dlqService;
    
    @Autowired
    private DLQAnalysisService analysisService;
    
    // 일정 시간 후 자동 재처리 (일시적 오류 가정)
    @Scheduled(fixedRate = 300000)  // 5분마다
    public void autoReprocessTemporaryErrors() {
        // 최근 1시간 내 실패한 메시지 중
        // 일시적 오류로 판단되는 메시지 자동 재처리
        LocalDateTime oneHourAgo = LocalDateTime.now().minusHours(1);
        
        List<DLQMessage> messages = dlqRepository.findByStatusAndFailedAtAfter(
            DLQStatus.PENDING, oneHourAgo
        );
        
        for (DLQMessage message : messages) {
            // 일시적 오류 판단 (예: Connection timeout, Network error)
            if (isTemporaryError(message)) {
                try {
                    dlqService.reprocessDLQMessage(message.getId());
                    log.info("Auto-reprocessed DLQ message: {}", message.getId());
                } catch (Exception e) {
                    log.error("Auto-reprocess failed: {}", message.getId(), e);
                }
            }
        }
    }
    
    private boolean isTemporaryError(DLQMessage message) {
        String errorType = message.getErrorType();
        String errorMessage = message.getErrorMessage();
        
        // 일시적 오류로 판단되는 패턴
        return errorType.contains("TimeoutException") ||
               errorType.contains("ConnectException") ||
               errorType.contains("SocketException") ||
               errorMessage.contains("connection") ||
               errorMessage.contains("timeout");
    }
}
```

### 5.3 수동 재처리 API

**운영자를 위한 재처리 API:**

```java
@RestController
@RequestMapping("/api/dlq")
@Slf4j
public class DLQController {
    
    @Autowired
    private DLQService dlqService;
    
    @Autowired
    private DLQAnalysisService analysisService;
    
    // DLQ 메시지 조회
    @GetMapping("/messages")
    public Page<DLQMessage> getDLQMessages(
            @RequestParam(required = false) String topic,
            @RequestParam(required = false) @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME) LocalDateTime from,
            @RequestParam(required = false) @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME) LocalDateTime to,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        
        Pageable pageable = PageRequest.of(page, size);
        
        if (topic != null && from != null && to != null) {
            return dlqService.getDLQMessages(topic, from, to, pageable);
        }
        
        return dlqService.getAllDLQMessages(pageable);
    }
    
    // DLQ 메시지 분석
    @GetMapping("/analysis")
    public DLQAnalysisResult analyzeDLQMessages(
            @RequestParam String topic,
            @RequestParam @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME) LocalDateTime from,
            @RequestParam @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME) LocalDateTime to) {
        
        return analysisService.analyzeDLQMessages(topic, from, to);
    }
    
    // DLQ 메시지 재처리
    @PostMapping("/messages/{id}/reprocess")
    public ResponseEntity<String> reprocessDLQMessage(@PathVariable Long id) {
        try {
            dlqService.reprocessDLQMessage(id);
            return ResponseEntity.ok("DLQ message reprocessed successfully");
        } catch (Exception e) {
            log.error("Failed to reprocess DLQ message: {}", id, e);
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body("Failed to reprocess: " + e.getMessage());
        }
    }
    
    // DLQ 메시지 일괄 재처리
    @PostMapping("/messages/batch-reprocess")
    public ResponseEntity<String> batchReprocessDLQMessages(
            @RequestBody List<Long> ids) {
        try {
            dlqService.reprocessDLQMessages(ids);
            return ResponseEntity.ok("Batch reprocess completed");
        } catch (Exception e) {
            log.error("Failed to batch reprocess DLQ messages", e);
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body("Failed to batch reprocess: " + e.getMessage());
        }
    }
    
    // DLQ 메시지 삭제
    @DeleteMapping("/messages/{id}")
    public ResponseEntity<String> deleteDLQMessage(@PathVariable Long id) {
        try {
            dlqService.deleteDLQMessage(id);
            return ResponseEntity.ok("DLQ message deleted");
        } catch (Exception e) {
            log.error("Failed to delete DLQ message: {}", id, e);
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body("Failed to delete: " + e.getMessage());
        }
    }
}
```

---

## 6. DLQ 운영 Best Practices

### 6.1 DLQ 설계 원칙

**1. 토픽별 DLQ 분리**

- 각 원본 토픽마다 별도의 DLQ 토픽 생성
- 토픽별로 다른 재처리 전략 적용 가능

**2. 메시지 보존 기간 설정**

- 중요 메시지는 오래 보관 (예: 30일)
- 일반 메시지는 짧게 보관 (예: 7일)
- 토픽별로 다른 retention 정책 적용

**3. 파티션 수 설정**

- 원본 토픽과 동일한 파티션 수 권장
- 키 기반 라우팅 일관성 유지

### 6.2 재시도 전략

**재시도 횟수 설정:**

- **일시적 오류**: 3-5번 재시도
- **비즈니스 로직 오류**: 즉시 DLQ로 이동
- **외부 서비스 오류**: 지수 백오프 적용

**재시도 간격:**

- Exponential backoff 권장
- 최대 재시도 간격 제한 (예: 10초)

### 6.3 모니터링 체크리스트

**필수 모니터링 항목:**

1. ✅ Consumer DLQ 메시지 수 (토픽별)
2. ✅ Consumer DLQ 메시지 비율 (전체 대비)
3. ✅ Consumer DLQ 재처리 성공률
4. ✅ Producer DLQ 메시지 수 (토픽별)
5. ✅ Producer 전송 실패율
6. ✅ 에러 유형별 분포
7. ✅ DLQ 메시지 보존 기간

**알림 임계값:**

**Consumer DLQ:**
- Consumer DLQ 메시지 10개 이상: 경고
- Consumer DLQ 메시지 100개 이상: 긴급
- Consumer DLQ 재처리 실패율 50% 이상: 경고

**Producer DLQ:**
- Producer DLQ 메시지 10개 이상: 경고
- Producer DLQ 메시지 100개 이상: 긴급
- Producer 전송 실패율 5% 이상: 경고
- Producer 전송 실패율 20% 이상: 긴급

### 6.4 재처리 전략

**재처리 전 확인사항:**

1. ✅ 원인 분석 완료 여부
2. ✅ 문제 해결 여부
3. ✅ 재처리 시 부작용 확인
4. ✅ 재처리 순서 확인 (순서가 중요한 경우)

**재처리 방법:**

- **즉시 재처리**: 문제 해결 후 즉시 재처리
- **지연 재처리**: 일정 시간 후 자동 재처리 (일시적 오류 가정)
- **수동 재처리**: 운영자가 선택적으로 재처리

---

## 마무리

**핵심 포인트:**

- **DLQ는 실패한 메시지를 보존하고 재처리할 수 있게 해주는 필수적인 메커니즘입니다.**
- **Consumer DLQ**: Spring Kafka의 `@RetryableTopic`을 사용하면 자동으로 DLQ를 관리할 수 있습니다.
- **Producer DLQ**: Spring Kafka가 자동으로 제공하지 않으므로, Callback 기반으로 수동 구현이 필요합니다.
- **DLQ 메시지 모니터링과 알림을 통해 문제를 조기에 발견하고 대응할 수 있습니다.**
- **DLQ 메시지 분석을 통해 근본 원인을 파악하고 재발 방지 전략을 수립할 수 있습니다.**

**최종 권장사항:**

- **Consumer DLQ**: Spring Kafka 3.3.11의 `@RetryableTopic` 활용 (자동화 권장)
- **Producer DLQ**: Callback 기반 수동 구현 (세밀한 제어 가능)
- **프로덕션 환경**: Consumer DLQ 자동 처리 + Producer DLQ 수동 처리 + 모니터링 + 알림 + 수동 재처리 API
- **초기 개발**: Spring Kafka 3.3.11의 `@RetryableTopic` 활용 (Consumer), Callback 기반 Producer DLQ 구현
- **고급 운영**: DLQ 메시지 분석 + 자동 재처리 + 대시보드

**Spring Kafka 3.3.11 사용 시 주의사항:**

- `@RetryableTopic`은 Spring Kafka 2.7+에서 도입되었으며, 3.3.11에서도 안정적으로 지원됩니다.
- `autoCreateTopics = "true"` 설정 시 자동으로 재시도 토픽과 DLQ 토픽이 생성되지만, 프로덕션 환경에서는 사전에 토픽을 생성하는 것을 권장합니다.
- `@DltHandler` 메서드는 반드시 같은 클래스 내에 있어야 하며, 원본 메서드와 동일한 파라미터 타입을 사용할 수 있습니다.
- **Producer DLQ는 Spring Kafka가 자동으로 제공하지 않으므로, Callback을 통한 수동 구현이 필요합니다.**

Kafka를 사용한 분산 시스템에서 **DLQ는 메시지 손실을 방지하고 시스템 안정성을 보장하는 핵심 메커니즘**입니다. Spring Kafka 3.3.11의 `@RetryableTopic`을 활용하면 재시도와 DLQ 처리를 자동화할 수 있으며, 일시적 오류는 자동으로 복구하고 영구적 오류는 수동으로 처리할 수 있습니다.

분산 시스템에서 메시지 처리뿐만 아니라 **캐시 관리**에서도 동시성 문제가 발생할 수 있습니다. 특히 **캐시 스탬피드(Cache Stampede)**는 동시에 많은 요청이 캐시 미스를 발생시켜 시스템에 큰 부하를 주는 문제입니다. 다음 글에서는 Redis를 활용한 캐시 스탬피드 방지 전략을 다뤄보겠습니다. 🚀
