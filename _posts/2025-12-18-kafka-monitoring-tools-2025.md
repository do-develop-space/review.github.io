---
layout: post
title: "2025년 Kafka 모니터링 도구 가이드: Producer와 Consumer 메트릭까지"
date: 2025-12-18
categories: [kafka, monitoring]
tags: [Kafka, 모니터링, Prometheus, Grafana, Confluent, KafkaManager, JMX, Producer, Consumer]
---

# 2025년 Kafka 모니터링 도구 가이드: Producer와 Consumer 메트릭까지

Kafka 클러스터를 운영할 때는 **Producer(발행)와 Consumer(소비) 메트릭을 모두 모니터링**해야 합니다.  
2025년 현재, 다양한 모니터링 도구들이 있지만 각각의 특징과 장단점을 이해하고 선택하는 것이 중요합니다.

이 글에서는 2025년 기준으로 Kafka 모니터링에 사용할 수 있는 주요 도구들과, Producer/Consumer 메트릭을 모두 모니터링하는 방법을 정리해보겠습니다.

---

## 1. 모니터링해야 할 메트릭

### Producer 메트릭 (발행 측)

**핵심 메트릭:**
- **메시지 전송 속도**: 초당 전송 메시지 수
- **전송 실패율**: 전송 실패한 메시지 비율
- **전송 지연 시간**: 메시지 전송에 걸리는 시간
- **전송 큐 크기**: 전송 대기 중인 메시지 수
- **재시도 횟수**: 실패 후 재시도한 횟수

**JMX 메트릭:**
```
kafka.producer:type=producer-metrics,client-id=<client-id>
  - record-send-rate
  - record-error-rate
  - record-retry-rate
  - request-latency-avg
  - buffer-available-bytes
```

### Consumer 메트릭 (소비 측)

**핵심 메트릭:**
- **Consumer Lag**: 처리되지 않은 메시지 수 (가장 중요!)
- **처리 속도**: 초당 처리 메시지 수
- **처리 지연 시간**: 메시지 처리에 걸리는 시간
- **컨슈머 그룹 상태**: 컨슈머 그룹의 활성 상태
- **리밸런싱 횟수**: 파티션 재분배 횟수

**JMX 메트릭:**
```
kafka.consumer:type=consumer-fetch-manager-metrics,client-id=<client-id>
  - records-lag-max
  - records-consumed-rate
  - fetch-latency-avg
  - fetch-rate
```

### Broker 메트릭

**클러스터 상태:**
- **브로커 상태**: 브로커의 활성/비활성 상태
- **토픽 파티션 수**: 각 토픽의 파티션 수
- **리더/팔로워 분산**: 파티션 리더 분산 상태
- **디스크 사용률**: 로그 세그먼트 디스크 사용량

---

## 2. 모니터링 도구 비교 (2025년 기준)

### 도구 비교표

| 도구 | 유형 | Producer 메트릭 | Consumer 메트릭 | 비용 | 난이도 |
|------|------|----------------|----------------|------|--------|
| **Prometheus + Grafana** | 오픈소스 | ✅ | ✅ | 무료 | 중간 |
| **Confluent Control Center** | 상용/오픈소스 | ✅ | ✅ | 유료(상용) | 쉬움 |
| **Kafka Manager (CMAK)** | 오픈소스 | ⚠️ 제한적 | ✅ | 무료 | 쉬움 |
| **Kafka UI** | 오픈소스 | ⚠️ 제한적 | ✅ | 무료 | 쉬움 |
| **Datadog** | 상용 SaaS | ✅ | ✅ | 유료 | 쉬움 |
| **New Relic** | 상용 SaaS | ✅ | ✅ | 유료 | 쉬움 |

---

## 3. Prometheus + Grafana (가장 권장)

### 개요

**Prometheus + Grafana**는 2025년 현재 가장 널리 사용되는 오픈소스 모니터링 스택입니다.

**장점:**
- ✅ Producer와 Consumer 메트릭 모두 모니터링 가능
- ✅ 완전 무료 (오픈소스)
- ✅ 커스터마이징 가능
- ✅ 다양한 대시보드 템플릿 제공
- ✅ 알림 기능 (Alertmanager)

**단점:**
- ⚠️ 설정이 복잡할 수 있음
- ⚠️ 직접 구축 및 운영 필요

### 설치 및 설정

**1. JMX Exporter 설정**

Kafka Broker와 Producer/Consumer 애플리케이션에 JMX Exporter를 추가:

```yaml
# jmx_prometheus_javaagent.jar 다운로드
# https://github.com/prometheus/jmx_exporter

# Kafka Broker 설정
KAFKA_OPTS="-javaagent:/path/to/jmx_prometheus_javaagent.jar=9999:/path/to/kafka-jmx-config.yml"
```

**JMX 설정 파일 (kafka-jmx-config.yml):**
```yaml
rules:
  # Broker 메트릭
  - pattern: kafka.server<type=(.+), name=(.+)><>Value
    name: kafka_server_$1_$2
  
  # Producer 메트릭
  - pattern: kafka.producer<type=(.+), client-id=(.+)><>Value
    name: kafka_producer_$1_$2
  
  # Consumer 메트릭
  - pattern: kafka.consumer<type=(.+), client-id=(.+)><>Value
    name: kafka_consumer_$1_$2
```

**2. Prometheus 설정**

```yaml
# prometheus.yml
scrape_configs:
  # Kafka Broker
  - job_name: 'kafka-broker'
    static_configs:
      - targets: ['broker1:9999', 'broker2:9999', 'broker3:9999']
  
  # Producer 애플리케이션
  - job_name: 'kafka-producer'
    static_configs:
      - targets: ['producer-app1:9999', 'producer-app2:9999']
  
  # Consumer 애플리케이션
  - job_name: 'kafka-consumer'
    static_configs:
      - targets: ['consumer-app1:9999', 'consumer-app2:9999']
```

**3. Grafana 대시보드**

**추천 대시보드:**
- **Kafka Overview**: [Grafana Dashboard ID 721](https://grafana.com/grafana/dashboards/721)
- **Kafka Producer**: [Grafana Dashboard ID 12311](https://grafana.com/grafana/dashboards/12311)
- **Kafka Consumer**: [Grafana Dashboard ID 12312](https://grafana.com/grafana/dashboards/12312)

**Producer 메트릭 쿼리 예시:**
```promql
# 초당 전송 메시지 수
rate(kafka_producer_producer_metrics_record_send_total[5m])

# 전송 실패율
rate(kafka_producer_producer_metrics_record_error_total[5m])

# 전송 지연 시간
kafka_producer_producer_metrics_request_latency_avg
```

**Consumer 메트릭 쿼리 예시:**
```promql
# Consumer Lag (가장 중요!)
kafka_consumer_consumer_fetch_manager_metrics_records_lag_max

# 초당 처리 메시지 수
rate(kafka_consumer_consumer_fetch_manager_metrics_records_consumed_total[5m])

# 처리 지연 시간
kafka_consumer_consumer_fetch_manager_metrics_fetch_latency_avg
```

### Producer/Consumer 메트릭 확인

**✅ Producer 메트릭 확인 가능:**
- 메시지 전송 속도
- 전송 실패율
- 전송 지연 시간
- 재시도 횟수

**✅ Consumer 메트릭 확인 가능:**
- Consumer Lag (실시간)
- 처리 속도
- 처리 지연 시간
- 컨슈머 그룹 상태

---

## 4. Confluent Control Center

### 개요

**Confluent Control Center**는 Confluent에서 제공하는 Kafka 관리 및 모니터링 도구입니다.

**장점:**
- ✅ Producer와 Consumer 메트릭 모두 제공
- ✅ 사용하기 쉬운 UI
- ✅ 실시간 모니터링
- ✅ 알림 기능
- ✅ 토픽 관리 기능

**단점:**
- ⚠️ 상용 버전은 유료 (Confluent Platform)
- ⚠️ 오픈소스 버전은 기능 제한적

### 주요 기능

**1. Producer 모니터링:**
- 메시지 전송 속도
- 전송 실패율
- 전송 지연 시간
- Producer별 메트릭

**2. Consumer 모니터링:**
- Consumer Lag (실시간)
- 처리 속도
- 컨슈머 그룹 상태
- 파티션별 Lag

**3. 클러스터 모니터링:**
- 브로커 상태
- 토픽 상태
- 파티션 분산

### 설치

**Docker Compose 예시:**
```yaml
version: '3'
services:
  control-center:
    image: confluentinc/cp-enterprise-control-center:latest
    ports:
      - "9021:9021"
    environment:
      CONTROL_CENTER_BOOTSTRAP_SERVERS: broker:9092
      CONTROL_CENTER_REPLICATION_FACTOR: 1
```

**접속:**
- URL: `http://localhost:9021`
- Producer/Consumer 메트릭을 대시보드에서 확인 가능

---

## 5. Kafka Manager (CMAK)

### 개요

**Kafka Manager (CMAK - Cluster Manager for Apache Kafka)**는 LinkedIn에서 개발한 오픈소스 도구입니다.

**장점:**
- ✅ 무료 (오픈소스)
- ✅ 사용하기 쉬운 UI
- ✅ 토픽 관리 기능
- ✅ Consumer Lag 모니터링

**단점:**
- ⚠️ Producer 메트릭은 제한적
- ⚠️ Consumer 메트릭은 기본적인 것만 제공
- ⚠️ 개발이 중단된 상태 (Yahoo에서 유지보수 중)

### 주요 기능

**Consumer 메트릭:**
- ✅ Consumer Lag 확인 가능
- ✅ 컨슈머 그룹 상태
- ✅ 파티션별 Lag

**Producer 메트릭:**
- ⚠️ 제한적 (토픽별 메시지 수 정도만)

### 설치

```bash
# GitHub에서 다운로드
git clone https://github.com/yahoo/CMAK.git
cd CMAK

# 실행
./sbt clean dist
cd target/universal
unzip cmak-*.zip
cd cmak-*/
bin/cmak
```

**접속:**
- URL: `http://localhost:9000`
- Consumer Lag는 확인 가능하지만, Producer 메트릭은 제한적

---

## 6. Kafka UI

### 개요

**Kafka UI**는 최근에 인기를 얻고 있는 오픈소스 Kafka 관리 도구입니다.

**장점:**
- ✅ 무료 (오픈소스)
- ✅ 모던한 UI
- ✅ 활발한 개발
- ✅ Docker로 쉽게 설치

**단점:**
- ⚠️ Producer 메트릭은 제한적
- ⚠️ Consumer 메트릭은 기본적인 것만

### 주요 기능

**Consumer 메트릭:**
- ✅ Consumer Lag 확인
- ✅ 컨슈머 그룹 상태
- ✅ 파티션별 메시지 조회

**Producer 메트릭:**
- ⚠️ 제한적

### 설치

**Docker Compose:**
```yaml
version: '3'
services:
  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    ports:
      - "8080:8080"
    environment:
      KAFKA_CLUSTERS_0_NAME: local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: broker:9092
```

**접속:**
- URL: `http://localhost:8080`
- Consumer Lag는 확인 가능

---

## 7. 상용 SaaS 솔루션

### Datadog

**장점:**
- ✅ Producer와 Consumer 메트릭 모두 제공
- ✅ 사용하기 쉬운 UI
- ✅ 자동 알림
- ✅ 인프라 모니터링 통합

**단점:**
- ⚠️ 유료 (월 구독)
- ⚠️ 데이터 보관 기간 제한

**주요 기능:**
- Producer 메트릭: 전송 속도, 실패율, 지연 시간
- Consumer 메트릭: Consumer Lag, 처리 속도
- 대시보드 및 알림

### New Relic

**장점:**
- ✅ Producer와 Consumer 메트릭 모두 제공
- ✅ APM 통합
- ✅ 사용하기 쉬운 UI

**단점:**
- ⚠️ 유료
- ⚠️ 설정이 복잡할 수 있음

---

## 8. 2025년 권장 사항

### 소규모/중소규모 프로젝트

**권장: Prometheus + Grafana**
- ✅ Producer와 Consumer 메트릭 모두 모니터링 가능
- ✅ 무료
- ✅ 커스터마이징 가능

**설정 예시:**
```yaml
# docker-compose.yml
services:
  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
  
  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
```

### 대규모 프로젝트 / 엔터프라이즈

**옵션 1: Confluent Control Center (상용)**
- ✅ 완전한 기능
- ✅ 엔터프라이즈 지원
- ⚠️ 비용 발생

**옵션 2: Prometheus + Grafana + 커스텀 대시보드**
- ✅ 무료
- ✅ 완전한 커스터마이징
- ⚠️ 직접 구축 및 운영 필요

### 빠른 프로토타이핑 / 개발 환경

**권장: Kafka UI**
- ✅ 빠른 설치
- ✅ 기본적인 모니터링
- ⚠️ Producer 메트릭은 제한적

---

## 9. Producer/Consumer 메트릭 모니터링 가이드

### Producer 메트릭 모니터링

**1. 애플리케이션에 JMX Exporter 추가**

```java
// Spring Boot 애플리케이션
// build.gradle
dependencies {
    implementation 'io.micrometer:micrometer-registry-prometheus'
}

// application.yml
management:
  endpoints:
    web:
      exposure:
        include: prometheus
  metrics:
    export:
      prometheus:
        enabled: true
```

**2. Producer 메트릭 확인**

```promql
# Prometheus 쿼리
# 초당 전송 메시지 수
rate(kafka_producer_record_send_total[5m])

# 전송 실패율
rate(kafka_producer_record_error_total[5m]) / rate(kafka_producer_record_send_total[5m])

# 전송 지연 시간 (P99)
histogram_quantile(0.99, rate(kafka_producer_request_latency_seconds_bucket[5m]))
```

### Consumer 메트릭 모니터링

**1. Consumer Lag 모니터링 (가장 중요!)**

```promql
# Consumer Lag (최대값)
max(kafka_consumer_consumer_fetch_manager_metrics_records_lag_max) by (topic, partition, consumer_group)

# Consumer Lag 알림 (1000 이상이면 경고)
kafka_consumer_consumer_fetch_manager_metrics_records_lag_max > 1000
```

**2. 처리 속도 모니터링**

```promql
# 초당 처리 메시지 수
rate(kafka_consumer_consumer_fetch_manager_metrics_records_consumed_total[5m])

# 처리 지연 시간
kafka_consumer_consumer_fetch_manager_metrics_fetch_latency_avg
```

### 통합 대시보드 예시

**Grafana 대시보드 구성:**

1. **Producer 섹션:**
   - 메시지 전송 속도 (초당)
   - 전송 실패율 (%)
   - 전송 지연 시간 (P50, P95, P99)

2. **Consumer 섹션:**
   - Consumer Lag (실시간)
   - 처리 속도 (초당)
   - 처리 지연 시간

3. **Broker 섹션:**
   - 브로커 상태
   - 디스크 사용률
   - 네트워크 처리량

---

## 10. 알림 설정

### Prometheus Alertmanager

**Consumer Lag 알림:**
```yaml
# alerts.yml
groups:
  - name: kafka_alerts
    rules:
      - alert: HighConsumerLag
        expr: kafka_consumer_consumer_fetch_manager_metrics_records_lag_max > 10000
        for: 5m
        annotations:
          summary: "High Consumer Lag detected"
          description: "Consumer lag is {{ $value }} messages"
      
      - alert: ProducerHighErrorRate
        expr: rate(kafka_producer_record_error_total[5m]) > 0.1
        for: 5m
        annotations:
          summary: "High Producer Error Rate"
          description: "Producer error rate is {{ $value }}"
```

---

## 마무리

**핵심 포인트:**

- **2025년 권장 도구**: Prometheus + Grafana (Producer/Consumer 메트릭 모두 지원)
- **Producer 메트릭**: 전송 속도, 실패율, 지연 시간 모니터링 필수
- **Consumer 메트릭**: Consumer Lag 모니터링이 가장 중요
- **도구 선택**: 프로젝트 규모와 예산에 따라 선택

**도구별 Producer/Consumer 메트릭 지원:**

- ✅ **Prometheus + Grafana**: Producer/Consumer 모두 완벽 지원 (권장)
- ✅ **Confluent Control Center**: Producer/Consumer 모두 지원 (상용)
- ⚠️ **Kafka Manager/UI**: Consumer는 지원, Producer는 제한적

2025년 현재, **Producer와 Consumer 메트릭을 모두 모니터링**하려면 **Prometheus + Grafana**를 사용하는 것이 가장 좋은 선택입니다. 무료이면서도 강력한 기능을 제공하며, 커스터마이징도 자유롭습니다. 특히 **Consumer Lag**는 Kafka 운영에서 가장 중요한 메트릭이므로 반드시 모니터링해야 합니다. 🚀

다음 글에서는 Kafka의 **메시지 순서 보장**과 파티션 전략에 대해 정리해보겠습니다.

