---
layout: post
title: "Kafka와 Elasticsearch 연동 가이드: 실시간 데이터 파이프라인 구축"
date: 2025-12-03
categories: [architecture]
tags: [Kafka, Elasticsearch, 데이터파이프라인, 실시간처리, 로그수집, 검색엔진]
---

# Kafka와 Elasticsearch 연동 가이드: 실시간 데이터 파이프라인 구축

마이크로서비스 아키텍처에서 실시간 로그 수집, 이벤트 스트리밍, 검색 기능을 구현할 때 **Kafka**와 **Elasticsearch**의 연동은 매우 중요한 패턴입니다. Kafka는 대용량 이벤트 스트리밍을 처리하고, Elasticsearch는 실시간 검색과 분석을 제공합니다. 이번 포스트에서는 Kafka와 Elasticsearch를 연동하여 실시간 데이터 파이프라인을 구축하는 방법을 상세히 알아보겠습니다.

## Kafka와 Elasticsearch 연동의 필요성

### 왜 연동해야 할까?

**Kafka의 역할:**
- 대용량 이벤트 스트리밍 처리
- 여러 서비스 간 이벤트 전달
- 이벤트 히스토리 보관

**Elasticsearch의 역할:**
- 실시간 검색 및 분석
- 로그 집계 및 모니터링
- 시계열 데이터 분석

**연동의 이점:**
- Kafka의 이벤트를 Elasticsearch에 자동 인덱싱
- 실시간 검색 및 분석 가능
- 로그 수집 파이프라인 구축
- 이벤트 기반 검색 기능 구현

## 연동 방법 비교

Kafka와 Elasticsearch를 연동하는 방법은 크게 세 가지가 있습니다:

### 1. Kafka Connect (권장)

**장점:**
- 공식 지원 도구
- 설정이 간단함
- 확장 가능한 아키텍처
- 다양한 커넥터 제공

**단점:**
- 추가 인프라 필요
- 일부 커스터마이징 제한

### 2. Logstash

**장점:**
- 강력한 데이터 변환 기능
- 다양한 입력/출력 플러그인
- 필터링 및 파싱 기능

**단점:**
- 리소스 사용량이 큼
- 설정이 복잡할 수 있음

### 3. 직접 구현 (Consumer Application)

**장점:**
- 완전한 제어권
- 비즈니스 로직 통합 가능
- 커스터마이징 자유도 높음

**단점:**
- 개발 및 유지보수 부담
- 에러 처리 및 재시도 로직 직접 구현 필요

## 방법 1: Kafka Connect를 사용한 연동

### Kafka Connect란?

**Kafka Connect**는 Kafka와 외부 시스템 간 데이터를 주고받는 표준 프레임워크입니다. Source Connector와 Sink Connector를 제공하여 다양한 시스템과 연동할 수 있습니다.

### Elasticsearch Sink Connector 설정

#### 1. Connector 설치

```bash
# Confluent Platform 사용 시
confluent-hub install confluentinc/kafka-connect-elasticsearch:latest

# 또는 수동 다운로드
wget https://github.com/confluentinc/kafka-connect-elasticsearch/releases/download/v14.0.0/kafka-connect-elasticsearch-14.0.0.jar
```

#### 2. Connector 설정 파일

```json
{
  "name": "elasticsearch-sink",
  "config": {
    "connector.class": "io.confluent.connect.elasticsearch.ElasticsearchSinkConnector",
    "tasks.max": "1",
    "topics": "user-events,order-events,product-events",
    "connection.url": "http://localhost:9200",
    "type.name": "_doc",
    "key.ignore": "false",
    "schema.ignore": "true",
    "batch.size": "100",
    "max.buffered.records": "10000",
    "flush.timeout.ms": "10000",
    "max.in.flight.requests": "5",
    "behavior.on.null.values": "ignore",
    "behavior.on.malformed.documents": "warn",
    "errors.tolerance": "all",
    "errors.log.enable": "true",
    "errors.log.include.messages": "true"
  }
}
```

#### 3. Connector 등록

```bash
# REST API를 통한 등록
curl -X POST http://localhost:8083/connectors \
  -H "Content-Type: application/json" \
  -d @elasticsearch-sink-config.json

# Connector 상태 확인
curl http://localhost:8083/connectors/elasticsearch-sink/status
```

### 주요 설정 옵션 설명

- **`topics`**: Elasticsearch로 전송할 Kafka 토픽 목록
- **`connection.url`**: Elasticsearch 클러스터 URL
- **`batch.size`**: 한 번에 인덱싱할 문서 수
- **`flush.timeout.ms`**: 버퍼를 플러시하는 시간 간격
- **`errors.tolerance`**: 에러 처리 방식 (all, none)
- **`key.ignore`**: Kafka 메시지 키를 Elasticsearch ID로 사용할지 여부

## 방법 2: Logstash를 사용한 연동

### Logstash 설정

#### 1. Logstash 설치

```bash
# Docker 사용
docker pull docker.elastic.co/logstash/logstash:8.15.3
```

#### 2. Logstash 설정 파일

```ruby
input {
  kafka {
    bootstrap_servers => "localhost:9092"
    topics => ["user-events", "order-events"]
    consumer_threads => 3
    group_id => "logstash-consumer"
    codec => "json"
    decorate_events => true
  }
}

filter {
  # JSON 파싱
  json {
    source => "message"
  }
  
  # 날짜 파싱
  date {
    match => ["timestamp", "ISO8601"]
  }
  
  # 필드 추가/변경
  mutate {
    add_field => { "index_name" => "kafka-%{+YYYY.MM.dd}" }
    remove_field => ["@version", "host"]
  }
}

output {
  elasticsearch {
    hosts => ["localhost:9200"]
    index => "%{index_name}"
    document_id => "%{[@metadata][kafka][key]}"
    template_name => "kafka-template"
    template => "/etc/logstash/templates/kafka-template.json"
    template_overwrite => true
  }
  
  # 디버깅용 stdout
  stdout {
    codec => rubydebug
  }
}
```

#### 3. Logstash 실행

```bash
# 설정 파일로 실행
logstash -f /etc/logstash/conf.d/kafka-elasticsearch.conf

# 또는 Docker
docker run -d \
  -v $(pwd)/logstash.conf:/usr/share/logstash/pipeline/logstash.conf \
  -p 5044:5044 \
  docker.elastic.co/logstash/logstash:8.15.3
```

## 방법 3: 직접 구현 (Spring Boot 예시)

### Spring Boot 애플리케이션 구현

#### 1. 의존성 추가

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter'
    implementation 'org.springframework.kafka:spring-kafka'
    implementation 'org.springframework.boot:spring-boot-starter-data-elasticsearch'
    implementation 'org.elasticsearch.client:elasticsearch-rest-high-level-client'
}
```

#### 2. Kafka Consumer 및 Elasticsearch 설정

```java
@Configuration
@EnableKafka
public class KafkaElasticsearchConfig {

    @Value("${spring.kafka.bootstrap-servers}")
    private String bootstrapServers;

    @Value("${spring.elasticsearch.uris}")
    private String elasticsearchUrl;

    // Kafka Consumer Factory
    @Bean
    public ConsumerFactory<String, String> consumerFactory() {
        Map<String, Object> props = new HashMap<>();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "elasticsearch-consumer-group");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
        return new DefaultKafkaConsumerFactory<>(props);
    }

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, String> kafkaListenerContainerFactory() {
        ConcurrentKafkaListenerContainerFactory<String, String> factory = 
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory());
        factory.setConcurrency(3);
        return factory;
    }

    // Elasticsearch Client
    @Bean
    public RestHighLevelClient elasticsearchClient() {
        HttpHost host = HttpHost.create(elasticsearchUrl);
        RestClientBuilder builder = RestClient.builder(host);
        return new RestHighLevelClient(builder);
    }
}
```

#### 3. Kafka Consumer 및 Elasticsearch 인덱싱

```java
@Service
@Slf4j
public class KafkaElasticsearchService {

    private final RestHighLevelClient elasticsearchClient;
    private final ObjectMapper objectMapper;

    public KafkaElasticsearchService(
            RestHighLevelClient elasticsearchClient,
            ObjectMapper objectMapper) {
        this.elasticsearchClient = elasticsearchClient;
        this.objectMapper = objectMapper;
    }

    @KafkaListener(topics = {"user-events", "order-events"})
    public void consumeAndIndex(String message, 
                                @Header(KafkaHeaders.RECEIVED_TOPIC) String topic,
                                @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
                                @Header(KafkaHeaders.OFFSET) long offset) {
        try {
            // JSON 파싱
            Map<String, Object> document = objectMapper.readValue(message, Map.class);
            
            // 메타데이터 추가
            document.put("kafka_topic", topic);
            document.put("kafka_partition", partition);
            document.put("kafka_offset", offset);
            document.put("@timestamp", Instant.now().toString());

            // Elasticsearch 인덱스 이름 결정
            String indexName = determineIndexName(topic, document);

            // 인덱싱 요청 생성
            IndexRequest request = new IndexRequest(indexName)
                    .source(document)
                    .id(generateDocumentId(topic, partition, offset));

            // Bulk API 사용 (성능 최적화)
            IndexResponse response = elasticsearchClient.index(request, RequestOptions.DEFAULT);
            
            log.info("Indexed document to Elasticsearch: index={}, id={}, version={}", 
                    response.getIndex(), response.getId(), response.getVersion());

        } catch (Exception e) {
            log.error("Error processing Kafka message: topic={}, partition={}, offset={}", 
                    topic, partition, offset, e);
            // 에러 처리: Dead Letter Queue 또는 재시도 로직
            handleError(message, topic, partition, offset, e);
        }
    }

    private String determineIndexName(String topic, Map<String, Object> document) {
        // 날짜 기반 인덱스 이름 생성
        String date = LocalDate.now().format(DateTimeFormatter.ofPattern("yyyy.MM.dd"));
        return topic + "-" + date;
    }

    private String generateDocumentId(String topic, int partition, long offset) {
        return topic + "-" + partition + "-" + offset;
    }

    private void handleError(String message, String topic, int partition, 
                            long offset, Exception e) {
        // Dead Letter Queue로 전송하거나 재시도 로직 구현
        // 예: 별도의 에러 토픽으로 전송
    }
}
```

#### 4. Bulk 인덱싱 최적화

```java
@Service
@Slf4j
public class BulkIndexingService {

    private final RestHighLevelClient elasticsearchClient;
    private final List<IndexRequest> bulkRequests = new ArrayList<>();
    private final int BULK_SIZE = 1000;
    private final long FLUSH_INTERVAL_MS = 5000;
    private long lastFlushTime = System.currentTimeMillis();

    @KafkaListener(topics = {"user-events", "order-events"})
    public void consumeAndBuffer(String message, 
                                 @Header(KafkaHeaders.RECEIVED_TOPIC) String topic) {
        try {
            Map<String, Object> document = objectMapper.readValue(message, Map.class);
            document.put("kafka_topic", topic);
            document.put("@timestamp", Instant.now().toString());

            String indexName = topic + "-" + LocalDate.now()
                    .format(DateTimeFormatter.ofPattern("yyyy.MM.dd"));

            IndexRequest request = new IndexRequest(indexName).source(document);
            bulkRequests.add(request);

            // 버퍼가 가득 찼거나 시간 간격이 지나면 플러시
            if (bulkRequests.size() >= BULK_SIZE || 
                (System.currentTimeMillis() - lastFlushTime) > FLUSH_INTERVAL_MS) {
                flushBulk();
            }

        } catch (Exception e) {
            log.error("Error buffering message: topic={}", topic, e);
        }
    }

    @Scheduled(fixedDelay = 5000)
    public void scheduledFlush() {
        if (!bulkRequests.isEmpty()) {
            flushBulk();
        }
    }

    private void flushBulk() {
        if (bulkRequests.isEmpty()) {
            return;
        }

        try {
            BulkRequest bulkRequest = new BulkRequest();
            bulkRequests.forEach(bulkRequest::add);

            BulkResponse bulkResponse = elasticsearchClient.bulk(bulkRequest, RequestOptions.DEFAULT);

            if (bulkResponse.hasFailures()) {
                log.error("Bulk indexing had failures: {}", bulkResponse.buildFailureMessage());
            } else {
                log.info("Successfully indexed {} documents", bulkRequests.size());
            }

            bulkRequests.clear();
            lastFlushTime = System.currentTimeMillis();

        } catch (Exception e) {
            log.error("Error during bulk indexing", e);
            // 실패한 요청은 재시도 큐에 추가
        }
    }
}
```

## 데이터 모델링 및 인덱스 템플릿

### Elasticsearch 인덱스 템플릿

```json
{
  "index_patterns": ["kafka-*"],
  "template": {
    "settings": {
      "number_of_shards": 3,
      "number_of_replicas": 1,
      "index.refresh_interval": "5s",
      "index.mapping.total_fields.limit": 2000
    },
    "mappings": {
      "properties": {
        "@timestamp": {
          "type": "date"
        },
        "kafka_topic": {
          "type": "keyword"
        },
        "kafka_partition": {
          "type": "integer"
        },
        "kafka_offset": {
          "type": "long"
        },
        "message": {
          "type": "text",
          "analyzer": "nori"
        }
      }
    }
  }
}
```

### 템플릿 적용

```bash
curl -X PUT "localhost:9200/_index_template/kafka-template" \
  -H 'Content-Type: application/json' \
  -d @kafka-template.json
```

## 모니터링 및 트러블슈팅

### 1. Kafka Consumer Lag 모니터링

```java
@Service
public class ConsumerLagMonitor {

    @Autowired
    private KafkaConsumer<String, String> kafkaConsumer;

    @Scheduled(fixedDelay = 60000)
    public void monitorLag() {
        Map<TopicPartition, Long> endOffsets = kafkaConsumer.endOffsets(
            kafkaConsumer.assignment()
        );
        
        Map<TopicPartition, Long> beginningOffsets = kafkaConsumer.beginningOffsets(
            kafkaConsumer.assignment()
        );

        for (TopicPartition partition : kafkaConsumer.assignment()) {
            long lag = endOffsets.get(partition) - beginningOffsets.get(partition);
            log.info("Partition {} lag: {}", partition, lag);
            
            if (lag > 10000) {
                log.warn("High lag detected on partition {}: {}", partition, lag);
            }
        }
    }
}
```

### 2. Elasticsearch 인덱싱 성능 모니터링

```bash
# 인덱싱 통계 확인
curl "localhost:9200/_stats/indexing?pretty"

# 인덱스별 통계
curl "localhost:9200/kafka-*/_stats/indexing?pretty"

# 클러스터 헬스 확인
curl "localhost:9200/_cluster/health?pretty"
```

### 3. 일반적인 문제 및 해결 방법

#### 문제 1: 인덱싱 속도가 느림

**해결 방법:**
- Bulk API 사용
- 버퍼 크기 증가
- 인덱스 refresh_interval 조정
- Replica 수 감소 (인덱싱 중)

#### 문제 2: 메모리 부족

**해결 방법:**
- JVM 힙 크기 증가
- Bulk 크기 감소
- Consumer 수 조정

#### 문제 3: 데이터 중복

**해결 방법:**
- 문서 ID를 Kafka 메시지 키나 (topic-partition-offset) 조합으로 생성
- Idempotent Producer 사용

## 모범 사례

### 1. 날짜 기반 인덱스 관리

```java
// 인덱스 이름을 날짜로 생성하여 자동 관리
String indexName = topic + "-" + LocalDate.now()
    .format(DateTimeFormatter.ofPattern("yyyy.MM.dd"));
```

### 2. 에러 처리 및 재시도

```java
@RetryableTopic(
    attempts = "3",
    backoff = @Backoff(delay = 1000, multiplier = 2),
    dltStrategy = DltStrategy.FAIL_ON_ERROR
)
@KafkaListener(topics = "user-events")
public void consumeWithRetry(String message) {
    // 처리 로직
}
```

### 3. 스키마 관리

```java
// Avro 스키마 사용
@KafkaListener(topics = "user-events")
public void consumeAvro(GenericRecord record) {
    // 타입 안전한 데이터 처리
}
```

### 4. 보안 설정

```yaml
# Kafka 보안
spring:
  kafka:
    security:
      protocol: SASL_SSL
    sasl:
      mechanism: PLAIN
      jaas:
        config: org.apache.kafka.common.security.plain.PlainLoginModule required username="user" password="pass";

# Elasticsearch 보안
spring:
  elasticsearch:
    uris: https://localhost:9200
    username: elastic
    password: ${ELASTICSEARCH_PASSWORD}
```

## 실제 사용 사례: Yellow Store 프로젝트

**Yellow Store 프로젝트에서는 다음과 같은 파이프라인을 구축하는 방법을 공부하고 있으며, 적용할 예정입니다:**

1. **이벤트 수집**: 주문, 결제, 재고 변경 등의 이벤트를 Kafka로 전송
2. **실시간 인덱싱**: Kafka Connect를 사용하여 Elasticsearch에 자동 인덱싱
3. **검색 기능**: 사용자 검색, 주문 조회, 상품 검색 등에 Elasticsearch 활용
4. **모니터링**: 로그 수집 및 분석을 위한 ELK 스택 구성

**아키텍처:**
```
마이크로서비스 → Kafka → Kafka Connect → Elasticsearch → 검색 API
```

이를 통해 실시간 검색 기능과 로그 분석을 효율적으로 구현할 수 있습니다.

---

다음 포스트에서는 **BFF(Backend for Frontend) 패턴**에 대해 정리해보겠습니다. 모바일 앱과 웹 프론트엔드 각각에 특화된 백엔드를 어떻게 설계하고, API Gateway·MSA 구조와 어떤 차이가 있는지 비교해보면서 실무에서의 적용 포인트를 정리해볼 예정입니다. 함께 아키텍처를 한 단계씩 확장해봐요! 🔍


