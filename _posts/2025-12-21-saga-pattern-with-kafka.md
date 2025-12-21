---
layout: post
title: "사가 패턴과 보상 트랜잭션: Kafka 기반 분산 트랜잭션 설계"
date: 2025-12-21
categories: [architecture, microservices]
tags: [Saga, 보상트랜잭션, Kafka, 분산트랜잭션, 마이크로서비스, EventDriven]
---

# 사가 패턴과 보상 트랜잭션: Kafka 기반 분산 트랜잭션 설계

마이크로서비스 아키텍처에서 하나의 비즈니스 기능(예: 주문 생성, 결제, 재고 차감)은 여러 서비스에 걸쳐 나뉘어 있습니다.  
이때 **모든 서비스의 작업을 한 번에 커밋/롤백하는 전통적인 트랜잭션(2PC)**을 쓰기 어렵기 때문에, **사가(Saga) 패턴**과 **보상 트랜잭션(Compensation Transaction)**이 등장했습니다.

이 글에서는 **Kafka를 메시지 브로커로 사용하는 것을 기준으로**, 사가 패턴과 보상 트랜잭션을 어떻게 설계할 수 있는지 정리해보겠습니다.

---

## 1. 왜 분산 트랜잭션이 어려운가?

### 단일 DB 트랜잭션과의 차이

단일 모놀리식 애플리케이션에서는 보통 하나의 데이터베이스에 대해 트랜잭션을 걸면 됩니다:

```text
[Application]
   ↓ (단일 트랜잭션)
[Single Database]
```

하지만 마이크로서비스에서는 서비스마다 **각자의 DB**를 가지는 것이 일반적입니다:

```text
[주문 서비스 DB]   [결제 서비스 DB]   [재고 서비스 DB]
```

이 경우, **하나의 ACID 트랜잭션으로 묶기 어렵고**, 2PC는 다음과 같은 이유로 잘 사용되지 않습니다:

- 구현 복잡도 증가
- 성능 저하 및 락 증가
- 장애 시 복구가 어려움

### 대안: 최종 일관성(Eventual Consistency)

분산 환경에서는 즉시 일관성 대신 **최종 일관성**을 선택하는 경우가 많습니다:

- 각 서비스는 **자기 DB에 대해서만 트랜잭션**을 사용
- 서비스 간에는 **이벤트(Event)**를 통해 상태를 전달
- 전체 시스템은 시간이 지나면서 일관된 상태로 수렴

여기서 **사가 패턴**이 등장합니다.

---

## 2. 사가(Saga) 패턴이란?

### 기본 개념

**사가(Saga)**는 하나의 비즈니스 프로세스를 여러 개의 **로컬 트랜잭션(local transaction)**으로 나누고,  
각 단계가 실패했을 때 이전 단계들을 취소하기 위해 **보상 트랜잭션(Compensation Transaction)**을 실행하는 패턴입니다.

```text
T1 → T2 → T3

T2에서 실패하면: C1 실행 (T1에 대한 보상)
T3에서 실패하면: C2, C1 순서로 보상 실행
```

- `T1, T2, T3`: 순방향 트랜잭션(정상 비즈니스 처리)
- `C1, C2`: 보상 트랜잭션(역방향 취소 처리)

### 예시: 주문-결제-재고 시나리오

1. **T1 (주문 서비스)**: 주문 생성 (`OrderCreated`)
2. **T2 (결제 서비스)**: 결제 승인 (`PaymentApproved`)
3. **T3 (재고 서비스)**: 재고 차감 (`InventoryReserved`)

**실패 시 보상 트랜잭션:**
- T2 실패 → `C1`: 주문 취소
- T3 실패 → `C2`: 결제 취소, `C1`: 주문 취소

---

## 3. 사가 패턴의 두 가지 스타일

### 1) 코레오그래피(Choreography) 방식

- 각 서비스가 **이벤트를 구독/발행**하며, 스스로 다음 액션을 결정
- 중앙 오케스트레이터 없이, 서비스들이 "춤추듯" 협력

```text
[Order Service] --OrderCreated--> [Kafka Topic]
    ↓                                  ↓
[Payment Service] --PaymentApproved--> [Kafka Topic]
    ↓                                  ↓
[Inventory Service] --InventoryReserved--> [Kafka Topic]
```

**장점:**
- 중앙 조정자 없음 (단순 구조)
- 서비스 간 결합도 낮음

**단점:**
- 흐름이 분산되어 있어 **전반적인 프로세스 파악이 어려움**
- 복잡한 비즈니스 로직에서는 이벤트 폭발(Event Storming) 발생 가능

### 2) 오케스트레이션(Orchestration) 방식

- 중앙 **사가 오케스트레이터(Saga Orchestrator)**가 전체 프로세스를 관리
- 각 서비스는 오케스트레이터의 명령을 수행

```text
[Saga Orchestrator]
    ↓  (Command)
[Order Service]
    ↓  (Event)
[Saga Orchestrator]
    ↓  (Command)
[Payment Service]
    ↓  (Event)
[Saga Orchestrator]
    ↓  (Command)
[Inventory Service]
```

**장점:**
- 흐름이 중앙에 모여 있어 **이해와 관리가 쉬움**
- 복잡한 분기/재시도 로직을 한 곳에서 관리 가능

**단점:**
- 오케스트레이터에 **로직이 집중되면서 비대해질 수 있음**
- 일부 중앙 집중화 구조

---

## 4. Kafka 기반 사가 설계

이 글에서는 **Kafka를 메시지 브로커로 사용하는 것을 기준**으로 설명합니다.

### 토픽 설계 예시

```text
order-created           # 주문 생성 이벤트
payment-approved        # 결제 승인 이벤트
payment-failed          # 결제 실패 이벤트
inventory-reserved      # 재고 예약 완료 이벤트
inventory-reservation-failed  # 재고 예약 실패 이벤트
order-cancelled         # 주문 취소 이벤트 (보상)
refund-processed        # 환불 완료 이벤트 (보상)
```

### 코레오그래피 예시 (Kafka)

**1) 주문 서비스 (Order Service)**

```java
// 주문 생성 후 이벤트 발행
@Transactional
public Order createOrder(CreateOrderCommand cmd) {
    Order order = orderRepository.save(Order.create(cmd));
    
    // Kafka에 OrderCreated 이벤트 발행
    OrderCreatedEvent event = OrderCreatedEvent.from(order);
    kafkaTemplate.send("order-created", event.getOrderId().toString(), event);
    
    return order;
}
```

**2) 결제 서비스 (Payment Service)**

```java
@KafkaListener(topics = "order-created", groupId = "payment-service")
@Transactional
public void handleOrderCreated(OrderCreatedEvent event) {
    try {
        Payment payment = paymentProcessor.approve(event.getOrderId(), event.getTotalAmount());
        paymentRepository.save(payment);
        
        PaymentApprovedEvent approved = PaymentApprovedEvent.from(payment);
        kafkaTemplate.send("payment-approved", event.getOrderId().toString(), approved);
    } catch (PaymentException ex) {
        PaymentFailedEvent failed = new PaymentFailedEvent(event.getOrderId(), ex.getMessage());
        kafkaTemplate.send("payment-failed", event.getOrderId().toString(), failed);
    }
}
```

**3) 재고 서비스 (Inventory Service)**

```java
@KafkaListener(topics = "payment-approved", groupId = "inventory-service")
@Transactional
public void handlePaymentApproved(PaymentApprovedEvent event) {
    try {
        inventoryService.reserve(event.getOrderId(), event.getItems());
        InventoryReservedEvent reserved = InventoryReservedEvent.from(event);
        kafkaTemplate.send("inventory-reserved", event.getOrderId().toString(), reserved);
    } catch (OutOfStockException ex) {
        InventoryReservationFailedEvent failed = new InventoryReservationFailedEvent(
            event.getOrderId(), ex.getMessage()
        );
        kafkaTemplate.send("inventory-reservation-failed", event.getOrderId().toString(), failed);
    }
}
```

---

## 5. 보상 트랜잭션 설계 (Compensation Transaction)

사가 패턴의 핵심은 **실패 시 이전 단계들을 되돌리는 보상 트랜잭션**입니다.

### 보상 트랜잭션 흐름 예시

**시나리오:**
1. 주문 생성 성공 (`OrderCreated`)
2. 결제 승인 성공 (`PaymentApproved`)
3. 재고 차감 실패 (`InventoryReservationFailed`)

**보상 흐름:**
1. 결제 취소 (`PaymentCancelled` / 환불 처리)
2. 주문 취소 (`OrderCancelled`)

### Kafka 이벤트 기반 보상 트랜잭션

**1) 재고 서비스: 실패 이벤트 발행**

```java
catch (OutOfStockException ex) {
    InventoryReservationFailedEvent failed = new InventoryReservationFailedEvent(
        event.getOrderId(), ex.getMessage()
    );
    kafkaTemplate.send("inventory-reservation-failed", event.getOrderId().toString(), failed);
}
```

**2) 결제 서비스: 환불(보상) 처리**

```java
@KafkaListener(topics = "inventory-reservation-failed", groupId = "payment-service")
@Transactional
public void handleInventoryReservationFailed(InventoryReservationFailedEvent event) {
    Payment payment = paymentRepository.findByOrderId(event.getOrderId())
        .orElseThrow(() -> new IllegalStateException("Payment not found"));
    
    payment.refund();
    paymentRepository.save(payment);
    
    RefundProcessedEvent refundEvent = RefundProcessedEvent.from(payment);
    kafkaTemplate.send("refund-processed", event.getOrderId().toString(), refundEvent);
}
```

**3) 주문 서비스: 주문 취소(보상) 처리**

```java
@KafkaListener(topics = {"payment-failed", "refund-processed"}, groupId = "order-service")
@Transactional
public void handleCompensation(CompensationEvent event) {
    Order order = orderRepository.findById(event.getOrderId())
        .orElseThrow(() -> new IllegalStateException("Order not found"));
    
    order.cancel("COMPENSATION: " + event.getReason());
    orderRepository.save(order);
    
    OrderCancelledEvent cancelled = OrderCancelledEvent.from(order);
    kafkaTemplate.send("order-cancelled", event.getOrderId().toString(), cancelled);
}
```

### 보상 트랜잭션 설계 시 주의사항

- **완전한 롤백이 아니라, 비즈니스적으로 취소하는 것**
  - 예: "재고 차감 취소"가 아니라, "환불 + 주문 취소"로 비즈니스 상태를 맞추는 것
- **보상 트랜잭션도 실패할 수 있음**
  - 예: 환불 API 실패 → 재시도 전략, 수동 개입(운영자 콘솔) 필요
- **보상은 순서를 거꾸로 실행**
  - T3 실패 → C2, C1 순서로 실행

---

## 6. 트랜잭션 경계와 Kafka

### 각 서비스 내부에서는 로컬 트랜잭션

각 서비스는 **자기 DB에 대해서만 @Transactional**을 사용합니다:

```java
@Transactional
public void handleOrderCreated(OrderCreatedEvent event) {
    // 1) 로컬 DB에 주문 저장
    // 2) Kafka에 다음 이벤트 발행
}
```

여기서 중요한 포인트는:
- DB 트랜잭션과 Kafka 메시지 발행 사이에 **일관성 문제가 생길 수 있다**는 것 (예: DB는 커밋됐는데 메시지가 안 나감)

### Outbox 패턴 (간단 개요)

이 문제를 해결하기 위해 **Outbox 패턴**을 사용할 수 있습니다:

1. 로컬 DB 트랜잭션 안에서 **비즈니스 데이터 + 이벤트 레코드(Outbox 테이블)**를 함께 저장
2. 별도 프로세스(또는 스레드)가 Outbox 테이블을 폴링하여 Kafka로 이벤트 발행
3. 발행이 성공하면 Outbox 레코드 삭제 또는 상태 업데이트

```text
[로컬 트랜잭션]
Order 테이블 INSERT
Outbox 테이블 INSERT (OrderCreated 이벤트)
→ COMMIT

[별도 프로세스]
Outbox SELECT → Kafka 전송 → 상태 DONE으로 업데이트
```

이 부분은 Outbox/Choreography 설계 글에서 더 자세히 다룰 수 있습니다.

---

## 7. 사가 패턴 vs 전통적 트랜잭션

### 비교

| 항목 | 전통적 트랜잭션(단일 DB) | 사가 패턴 |
|------|--------------------------|-----------|
| 범위 | 단일 DB | 여러 서비스 / 여러 DB |
| 일관성 | 즉시 일관성 | 최종 일관성 |
| 롤백 방식 | DB 롤백 | 보상 트랜잭션(비즈니스 롤백) |
| 복잡도 | 낮음 | 높음 (이벤트 설계, 보상 로직 필요) |
| 사용 사례 | 모놀리식, 단일 DB 시스템 | 마이크로서비스, 분산 시스템 |

### 언제 사가 패턴을 사용할까?

- **여러 서비스/DB를 가로지르는 비즈니스 프로세스**가 있을 때
- **강한 일관성보다 가용성과 확장성**이 더 중요한 시스템일 때
- 2PC(분산 트랜잭션)을 쓰기 어렵거나 원하지 않을 때

---

## 8. Kafka 기반 사가 설계 체크리스트

### 설계 체크리스트

- [ ] 비즈니스 프로세스를 **단계별 로컬 트랜잭션**으로 나눴는가?
- [ ] 각 단계의 **성공 이벤트/실패 이벤트**를 정의했는가?
- [ ] 실패 시 실행해야 할 **보상 트랜잭션**을 정의했는가?
- [ ] 이벤트 순서를 고려한 **토픽 설계**가 되어 있는가?
- [ ] 재시도/중복 처리(Idempotency) 전략이 있는가?
- [ ] Outbox 패턴 등으로 **DB와 Kafka 간 일관성**을 보장하는가?

### 운영 체크리스트

- [ ] 사가 진행 상태를 모니터링할 수 있는가? (예: 주문 상태: PENDING / COMPLETED / COMPENSATED)
- [ ] 보상 트랜잭션 실패 시 운영자가 개입할 수 있는 도구(대시보드, 관리 API)가 있는가?
- [ ] Kafka 토픽/컨슈머 그룹 상태를 모니터링하고 있는가? (Consumer Lag, 에러 로그 등)

---

## 마무리

**핵심 포인트:**

- **사가 패턴**은 분산 환경에서 하나의 비즈니스 프로세스를 여러 로컬 트랜잭션으로 나누어 처리하는 패턴입니다.
- **보상 트랜잭션**은 실패 시 이전 단계들을 비즈니스적으로 취소하는 역할을 합니다.
- **Kafka**를 사용하면 이벤트 기반으로 사가를 자연스럽게 구현할 수 있으며, 코레오그래피/오케스트레이션 두 가지 스타일을 선택할 수 있습니다.
- 각 서비스는 **자기 DB에 대해서만 @Transactional**을 사용하고, 서비스 간 일관성은 **이벤트와 보상 트랜잭션**으로 맞춥니다.

사가 패턴은 설계와 구현 난도가 높지만, 마이크로서비스에서 **확장성과 유연성을 유지하면서 분산 트랜잭션 문제를 해결하는 핵심 패턴**입니다.  
앞으로는 Outbox 패턴, Idempotency, 재시도 전략 등을 포함한 **실전 Kafka 사가 구현 예제**도 정리해볼 예정입니다. 🚀

