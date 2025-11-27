---
layout: post
title: "Kafka vs RabbitMQ: 메시지 브로커의 동작 차이와 선택 가이드"
date: 2025-11-27
categories: [architecture]
tags: [Kafka, RabbitMQ, 메시지브로커, 이벤트스트리밍, 메시지큐]
---

# Kafka vs RabbitMQ: 메시지 브로커의 동작 차이와 선택 가이드

마이크로서비스 아키텍처에서 서비스 간 비동기 통신을 구현할 때 가장 많이 고려되는 메시지 브로커가 바로 Kafka와 RabbitMQ입니다. 두 기술은 모두 메시지를 전달하는 역할을 하지만, 내부 아키텍처와 동작 방식이 근본적으로 다릅니다. 이번 포스트에서는 두 메시지 브로커의 차이점을 명확히 하고, 각각의 사용 시나리오를 알아보겠습니다.

## RabbitMQ란?

RabbitMQ는 **AMQP(Advanced Message Queuing Protocol)** 기반의 메시지 브로커입니다. 전통적인 메시지 큐 방식으로 동작하며, 메시지를 큐에 저장하고 소비자가 가져가는 방식입니다.

### RabbitMQ의 동작 방식

```
Producer → Exchange → Queue → Consumer
```

1. **Producer**: 메시지를 Exchange로 전송
2. **Exchange**: 라우팅 규칙에 따라 메시지를 적절한 Queue로 전달
3. **Queue**: 메시지를 저장 (소비자가 가져갈 때까지 보관)
4. **Consumer**: Queue에서 메시지를 가져와 처리

### RabbitMQ의 특징

**장점:**
- ✅ **유연한 라우팅**: Exchange 타입에 따른 다양한 라우팅 패턴
- ✅ **메시지 보장**: 메시지 영속성, 확인 응답(Acknowledgment) 지원
- ✅ **우선순위 큐**: 메시지 우선순위 설정 가능
- ✅ **TTL(Time To Live)**: 메시지 만료 시간 설정
- ✅ **Dead Letter Queue**: 처리 실패한 메시지 관리
- ✅ **관리 UI**: 웹 기반 관리 도구 제공

**단점:**
- ❌ **처리량 제한**: 초당 수만~수십만 메시지 처리
- ❌ **메시지 삭제**: 소비되면 큐에서 제거
- ❌ **확장성 제약**: 수평 확장이 제한적

### RabbitMQ 사용 예제

```java
// Producer
@Autowired
private RabbitTemplate rabbitTemplate;

public void sendMessage(String message) {
    rabbitTemplate.convertAndSend("exchange", "routing.key", message);
}

// Consumer
@RabbitListener(queues = "my.queue")
public void receiveMessage(String message) {
    System.out.println("Received: " + message);
    // 메시지 처리
}
```

## Kafka란?

Kafka는 **분산 이벤트 스트리밍 플랫폼**입니다. 로그 기반의 분산 메시지 시스템으로, 메시지를 토픽(Topic)에 저장하고 여러 소비자가 읽을 수 있는 방식입니다.

### Kafka의 동작 방식

```
Producer → Topic (Partitions) → Consumer Groups
```

1. **Producer**: 메시지를 Topic의 Partition에 전송
2. **Topic/Partition**: 메시지를 순서대로 저장 (로그 형태)
3. **Consumer Group**: 여러 Consumer가 같은 그룹에서 메시지를 분산 처리
4. **Offset**: 각 Consumer가 어디까지 읽었는지 추적

### Kafka의 특징

**장점:**
- ✅ **높은 처리량**: 초당 수백만~수천만 메시지 처리 가능
- ✅ **메시지 보존**: 설정한 기간 동안 메시지 보관 (삭제되지 않음)
- ✅ **여러 소비자**: 같은 메시지를 여러 Consumer Group이 읽을 수 있음
- ✅ **수평 확장**: Partition을 통한 쉬운 확장
- ✅ **이벤트 스트리밍**: 실시간 데이터 스트리밍에 최적화
- ✅ **내구성**: 분산 시스템으로 높은 가용성

**단점:**
- ❌ **복잡한 설정**: 운영 및 관리가 복잡
- ❌ **높은 리소스 사용**: 디스크와 메모리 사용량이 많음
- ❌ **단순한 라우팅**: Exchange 같은 복잡한 라우팅 없음
- ❌ **메시지 순서**: Partition 내에서만 순서 보장

### Kafka 사용 예제

```java
// Producer
@Autowired
private KafkaTemplate<String, String> kafkaTemplate;

public void sendMessage(String message) {
    kafkaTemplate.send("my-topic", message);
}

// Consumer
@KafkaListener(topics = "my-topic", groupId = "my-group")
public void consumeMessage(String message) {
    System.out.println("Received: " + message);
    // 메시지 처리
}
```

## 핵심 차이점

### 1. 메시지 저장 방식

**RabbitMQ:**
- 메시지를 **큐에 저장**
- 소비자가 가져가면 **메시지 삭제**
- 큐가 비어있으면 메시지 없음

**Kafka:**
- 메시지를 **로그 파일에 저장**
- 소비해도 **메시지가 삭제되지 않음**
- 설정한 기간(예: 7일) 동안 보관

### 2. 소비 패턴

**RabbitMQ:**
```
Queue: [Msg1, Msg2, Msg3]
Consumer1: Msg1 가져감 → Queue: [Msg2, Msg3]
Consumer2: Msg2 가져감 → Queue: [Msg3]
```

**Kafka:**
```
Topic Partition: [Msg1, Msg2, Msg3, ...]
Consumer Group A: Offset 0 → Msg1, Msg2 읽음 (Offset 2)
Consumer Group B: Offset 0 → Msg1, Msg2 읽음 (Offset 2)
(메시지는 삭제되지 않음)
```

### 3. 처리량

**RabbitMQ:**
- 초당 수만~수십만 메시지
- 단일 큐 기준

**Kafka:**
- 초당 수백만~수천만 메시지
- Partition 수에 비례하여 증가

### 4. 메시지 순서 보장

**RabbitMQ:**
- 큐 내에서 순서 보장
- 여러 큐로 분산 시 순서 보장 어려움

**Kafka:**
- Partition 내에서만 순서 보장
- 같은 Partition Key를 사용하면 순서 보장

### 5. 확장성

**RabbitMQ:**
- 클러스터링 지원
- 큐 단위로 분산
- 확장성이 제한적

**Kafka:**
- Partition을 통한 수평 확장
- 매우 높은 확장성
- 수백 개의 Broker로 확장 가능

## 사용 시나리오 비교

### RabbitMQ가 적합한 경우

✅ **작업 큐 (Task Queue)**
- 비동기 작업 처리
- 작업 완료 후 메시지 삭제
- 예: 이미지 처리, 이메일 발송

✅ **요청-응답 패턴**
- RPC 스타일 통신
- 메시지 교환 후 삭제
- 예: 서비스 간 동기화된 통신

✅ **복잡한 라우팅**
- 다양한 라우팅 규칙 필요
- Exchange를 통한 유연한 라우팅
- 예: 이벤트 기반 라우팅

✅ **낮은 처리량**
- 초당 수만~수십만 메시지
- 단순한 메시지 큐 요구사항

**예시:**
```java
// 주문 처리 후 이메일 발송 (작업 큐)
rabbitTemplate.convertAndSend("email.queue", emailMessage);
// 이메일 발송 후 메시지 삭제됨
```

### Kafka가 적합한 경우

✅ **이벤트 스트리밍**
- 실시간 데이터 스트림 처리
- 여러 소비자가 같은 이벤트 읽기
- 예: 사용자 활동 로그, 클릭 스트림

✅ **높은 처리량**
- 초당 수백만~수천만 메시지
- 대용량 데이터 처리
- 예: IoT 센서 데이터, 로그 수집

✅ **이벤트 소싱**
- 모든 이벤트를 보관
- 이벤트 재생 가능
- 예: 주문 히스토리, 감사 로그

✅ **여러 소비자 패턴**
- 같은 메시지를 여러 서비스가 읽어야 함
- 메시지가 삭제되지 않아야 함
- 예: 주문 이벤트를 재고, 배송, 마케팅 서비스가 모두 읽기

**예시:**
```java
// 주문 생성 이벤트 (여러 서비스가 읽어야 함)
kafkaTemplate.send("order-created", orderEvent);
// 재고 서비스, 배송 서비스, 마케팅 서비스가 모두 읽음
// 메시지는 보관되어 나중에 재처리 가능
```

## 실제 아키텍처 예제

### RabbitMQ 아키텍처

```
[주문 서비스] → Exchange → [결제 큐] → [결제 서비스]
                      → [이메일 큐] → [이메일 서비스]
                      → [알림 큐] → [알림 서비스]
```

**특징:**
- 각 서비스가 메시지를 가져가면 삭제
- 작업 완료 후 메시지 불필요
- 단순한 비동기 작업 처리

### Kafka 아키텍처

```
[주문 서비스] → Topic: order-events
                    ↓
        [Consumer Group: inventory] → [재고 서비스]
        [Consumer Group: shipping] → [배송 서비스]
        [Consumer Group: analytics] → [분석 서비스]
```

**특징:**
- 모든 서비스가 같은 이벤트를 읽을 수 있음
- 이벤트가 보관되어 재처리 가능
- 이벤트 소싱 및 CQRS 패턴에 적합

## 성능 비교

### 처리량

```
RabbitMQ:
  - 단일 큐: ~20,000 msg/s
  - 클러스터: ~100,000 msg/s

Kafka:
  - 단일 Partition: ~100,000 msg/s
  - 여러 Partition: ~1,000,000+ msg/s
```

### 지연 시간

```
RabbitMQ:
  - 평균: 1-5ms
  - 낮은 지연 시간

Kafka:
  - 평균: 5-20ms
  - 디스크 쓰기로 인한 지연
```

## 함께 사용하기

일부 시스템에서는 두 기술을 함께 사용하기도 합니다:

- **RabbitMQ**: 작업 큐, 요청-응답 패턴
- **Kafka**: 이벤트 스트리밍, 로그 수집

```
[서비스] → RabbitMQ (작업 큐) → [워커 서비스]
        → Kafka (이벤트) → [이벤트 소싱]
```

## 결론

### RabbitMQ를 선택해야 하는 경우

✅ 작업 큐 및 비동기 작업 처리
✅ 요청-응답 패턴
✅ 복잡한 라우팅이 필요한 경우
✅ 낮은 처리량으로 충분한 경우
✅ 메시지가 소비되면 삭제되어도 되는 경우

### Kafka를 선택해야 하는 경우

✅ 이벤트 스트리밍 및 실시간 데이터 처리
✅ 매우 높은 처리량 필요
✅ 여러 소비자가 같은 메시지를 읽어야 하는 경우
✅ 이벤트 소싱 및 CQRS 패턴
✅ 메시지를 보관하고 재처리해야 하는 경우

**Yellow Store 프로젝트에서는 주문 이벤트를 여러 서비스(재고, 배송, 정산)가 모두 읽어야 하고, 이벤트 히스토리를 보관해야 하므로 Kafka를 선택하는 것이 적합합니다.** 하지만 단순한 비동기 작업(이메일 발송 등)은 RabbitMQ를 사용하는 것이 더 효율적일 수 있습니다.

---

다음 포스트에서는 **역직렬화가 필요한 이유**에 대해 다루겠습니다. 객체 직렬화와 역직렬화의 개념, 그리고 왜 필요한지 알아보겠습니다. 많은 관심 부탁드립니다! 🔄

