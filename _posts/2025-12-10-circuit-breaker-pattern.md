---
layout: post
title: "Circuit Breaker 패턴: 마이크로서비스 장애 전파를 막는 방법"
date: 2025-12-10
categories: [microservices]
tags: [Circuit Breaker, Hystrix, Resilience4j, Spring Cloud, 장애격리, 마이크로서비스]
---

# Circuit Breaker 패턴: 마이크로서비스 장애 전파를 막는 방법

마이크로서비스 아키텍처에서 **한 서비스의 장애가 다른 서비스로 전파되는 것(Cascading Failure)**을 막는 것은 매우 중요합니다.  
이를 위한 핵심 패턴 중 하나가 **Circuit Breaker(회로 차단기)** 패턴입니다.

이 글에서는 Circuit Breaker 패턴의 개념과 동작 원리, 그리고 Spring 기반 마이크로서비스에서 실제로 적용하는 방법을 정리해보겠습니다.

---

## 1. Circuit Breaker 패턴이란?

### 전기 회로의 누전차단기와 같은 원리

모든 전기 사용 장소에는 **누전차단기**가 있습니다. 누전차단기는 전기 사용 중 누전, 과전류, 합선으로 **전기사고가 발생하기 전에 전기를 미리 차단**하는 역할을 합니다.

마이크로서비스 아키텍처에서도 동일한 원리가 적용됩니다:

- **정상 상태**: 서비스 간 통신이 원활하게 이루어짐
- **장애 발생**: 특정 서비스가 느려지거나 응답하지 않음
- **장애 전파**: 문제가 있는 서비스를 호출하는 다른 서비스들도 계속 느려지거나 타임아웃 발생
- **전체 서비스 중단**: 도미노처럼 장애가 전 서비스에 번져 전체 시스템이 마비됨

Circuit Breaker는 **문제가 있는 서비스로의 트래픽을 차단**하여, 전체 서비스가 느려지거나 중단되는 것을 미리 방지하는 역할을 합니다.

---

## 2. Circuit Breaker의 동작 원리

Circuit Breaker는 **3가지 상태**를 가집니다:

### CLOSED (정상 상태)

- 정상적으로 요청을 전달하고 응답을 받음
- 실패율이나 응답 시간을 모니터링
- 정상적인 응답이 계속 오면 CLOSED 상태 유지

### OPEN (차단 상태)

- 일정 횟수 이상 비정상적인 응답(타임아웃, 에러 등)이 발생하면 **OPEN 상태로 전환**
- 더 이상 문제가 있는 서비스를 호출하지 않음
- **빠른 실패(Fast Failing)**를 위해 **Fallback 메서드**를 즉시 호출
- Fallback 메서드는 에러 메시지, 캐싱된 결과, 기본값 등을 반환

### HALF-OPEN (테스트 상태)

- OPEN 상태에서 일정 시간이 지나면, 서비스가 복구되었는지 확인하기 위해 **한 번의 요청을 시도**
- 정상적인 응답이 오면 → **CLOSED 상태로 전환**하고 정상 호출 재개
- 비정상적인 응답이 오면 → **OPEN 상태 유지**하고 타이머를 다시 초기화

---

## 3. Circuit Breaker 구현 라이브러리

### Hystrix (Netflix)

**과거의 표준이었지만 현재는 유지보수 모드**

- Netflix에서 개발한 Circuit Breaker 라이브러리
- Spring Cloud Netflix의 핵심 컴포넌트로 널리 사용됨
- 2018년부터 **유지보수 모드(Maintenance Mode)**로 전환
- 새로운 기능 추가는 중단되었지만, 기존 프로젝트에서는 여전히 사용 가능

**특징:**
- `@HystrixCommand` 어노테이션으로 간단하게 적용 가능
- Hystrix Dashboard로 모니터링 가능
- Thread Pool Isolation, Semaphore Isolation 지원

### Resilience4j (현재 권장)

**Hystrix의 현대적인 대안**

- Java 8+ 함수형 프로그래밍 기반
- Hystrix보다 가볍고, 모듈화된 설계
- Circuit Breaker 외에도 Rate Limiter, Retry, Bulkhead, TimeLimiter 등 다양한 패턴 제공

**특징:**
- Spring Boot와의 통합이 잘 되어 있음
- 메트릭을 Micrometer로 노출하여 Prometheus, Grafana와 연동 가능
- 함수형 인터페이스 기반으로 유연한 사용 가능

### Spring Cloud Circuit Breaker

**Spring의 공식 추상화 레이어**

- Circuit Breaker 구현체를 추상화한 Spring의 공식 프로젝트
- Resilience4j, Hystrix, Sentinel 등 다양한 구현체를 지원
- 구현체를 쉽게 교체할 수 있도록 추상화

**지원하는 구현체:**
- Spring Cloud Circuit Breaker Resilience4j (권장)
- Spring Cloud Circuit Breaker Hystrix (레거시)
- Spring Cloud Circuit Breaker Sentinel

---

## 4. Resilience4j를 사용한 실전 예제

### 의존성 추가

```gradle
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.cloud:spring-cloud-starter-circuitbreaker-resilience4j'
    implementation 'io.github.resilience4j:resilience4j-spring-boot3'
}
```

### 설정 파일 (application.yaml)

```yaml
resilience4j:
  circuitbreaker:
    instances:
      orderService:
        registerHealthIndicator: true
        slidingWindowSize: 10
        minimumNumberOfCalls: 5
        permittedNumberOfCallsInHalfOpenState: 3
        automaticTransitionFromOpenToHalfOpenEnabled: true
        waitDurationInOpenState: 10s
        failureRateThreshold: 50
        eventConsumerBufferSize: 10
        recordExceptions:
          - org.springframework.web.client.HttpServerErrorException
          - java.util.concurrent.TimeoutException
          - java.io.IOException
```

**설정 설명:**
- `slidingWindowSize`: 실패율 계산을 위한 슬라이딩 윈도우 크기
- `minimumNumberOfCalls`: Circuit Breaker가 동작하기 위한 최소 호출 횟수
- `waitDurationInOpenState`: OPEN 상태에서 HALF-OPEN으로 전환하기까지 대기 시간
- `failureRateThreshold`: 실패율 임계값 (50% = 50% 이상 실패 시 OPEN)
- `permittedNumberOfCallsInHalfOpenState`: HALF-OPEN 상태에서 허용할 호출 횟수

### 서비스 레이어 구현

```java
@Service
public class OrderService {
    
    private final RestTemplate restTemplate;
    private final CircuitBreaker circuitBreaker;
    
    public OrderService(RestTemplate restTemplate, 
                       CircuitBreakerRegistry circuitBreakerRegistry) {
        this.restTemplate = restTemplate;
        this.circuitBreaker = circuitBreakerRegistry.circuitBreaker("orderService");
    }
    
    public String getOrderInfo(String orderId) {
        return circuitBreaker.executeSupplier(() -> {
            // 외부 서비스 호출
            return restTemplate.getForObject(
                "http://order-service/api/orders/" + orderId, 
                String.class
            );
        });
    }
    
    // Fallback 메서드
    public String getOrderInfoFallback(String orderId, Exception ex) {
        return "주문 정보를 가져올 수 없습니다. 잠시 후 다시 시도해주세요.";
    }
}
```

### 어노테이션 기반 사용 (더 간단한 방법)

```java
@Service
public class OrderService {
    
    @CircuitBreaker(name = "orderService", fallbackMethod = "getOrderInfoFallback")
    public String getOrderInfo(String orderId) {
        return restTemplate.getForObject(
            "http://order-service/api/orders/" + orderId, 
            String.class
        );
    }
    
    public String getOrderInfoFallback(String orderId, Exception ex) {
        return "주문 정보를 가져올 수 없습니다. 잠시 후 다시 시도해주세요.";
    }
}
```

---

## 5. Fallback 전략 설계

Fallback은 Circuit Breaker가 OPEN 상태일 때 호출되는 대체 로직입니다. 효과적인 Fallback 전략을 수립하는 것이 중요합니다.

### 캐싱된 데이터 반환

```java
@CircuitBreaker(name = "productService", fallbackMethod = "getProductFallback")
public Product getProduct(String productId) {
    return restTemplate.getForObject(
        "http://product-service/api/products/" + productId, 
        Product.class
    );
}

public Product getProductFallback(String productId, Exception ex) {
    // 캐시에서 조회
    return cacheService.getProduct(productId)
        .orElse(Product.defaultProduct());
}
```

### 기본값 반환

```java
@CircuitBreaker(name = "inventoryService", fallbackMethod = "checkStockFallback")
public boolean checkStock(String productId, int quantity) {
    return restTemplate.getForObject(
        "http://inventory-service/api/stock/" + productId + "/" + quantity, 
        Boolean.class
    );
}

public boolean checkStockFallback(String productId, int quantity, Exception ex) {
    // 기본값: 재고가 있다고 가정 (비즈니스 로직에 따라 다름)
    return true;
}
```

### 에러 응답 반환

```java
@CircuitBreaker(name = "paymentService", fallbackMethod = "processPaymentFallback")
public PaymentResult processPayment(PaymentRequest request) {
    return restTemplate.postForObject(
        "http://payment-service/api/payments", 
        request, 
        PaymentResult.class
    );
}

public PaymentResult processPaymentFallback(PaymentRequest request, Exception ex) {
    return PaymentResult.failure("결제 서비스가 일시적으로 사용할 수 없습니다.");
}
```

---

## 6. 모니터링과 관찰 가능성

Circuit Breaker의 상태를 모니터링하는 것은 운영에서 매우 중요합니다.

### Actuator 엔드포인트

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,circuitbreakers,metrics
  metrics:
    export:
      prometheus:
        enabled: true
```

### Health Check

```
GET /actuator/health
```

응답 예시:
```json
{
  "status": "UP",
  "components": {
    "circuitBreakers": {
      "status": "UP",
      "details": {
        "orderService": {
          "status": "CLOSED",
          "details": {
            "failureRate": "0.0%",
            "slowCallRate": "0.0%"
          }
        }
      }
    }
  }
}
```

### Circuit Breaker 메트릭

```
GET /actuator/metrics/resilience4j.circuitbreaker.calls
```

---

## 7. 주의사항과 베스트 프랙티스

### 1. Fallback 로직은 빠르게 실행되어야 함

Fallback 자체가 느리면 Circuit Breaker의 의미가 없어집니다. Fallback은 동기적으로 빠르게 실행되거나, 비동기적으로 처리되어야 합니다.

### 2. 적절한 임계값 설정

- `failureRateThreshold`가 너무 낮으면 → 정상적인 일시적 오류에도 Circuit이 열림
- `failureRateThreshold`가 너무 높으면 → 실제 장애 상황을 감지하지 못함
- 트래픽 패턴과 서비스 특성에 맞게 튜닝 필요

### 3. 타임아웃 설정과 함께 사용

Circuit Breaker만으로는 부족합니다. **타임아웃(Timeout)** 설정과 함께 사용하여 응답이 없는 요청을 빠르게 실패 처리해야 합니다.

```yaml
resilience4j:
  timelimiter:
    instances:
      orderService:
        timeoutDuration: 2s
```

### 4. 서비스별로 독립적인 Circuit Breaker 인스턴스 사용

각 외부 서비스마다 별도의 Circuit Breaker 인스턴스를 구성하여, 한 서비스의 장애가 다른 서비스의 Circuit Breaker에 영향을 주지 않도록 해야 합니다.

### 5. 로깅과 알림 연동

Circuit Breaker가 OPEN 상태로 전환될 때는 **알림(Alert)**을 발송하여 운영팀이 즉시 대응할 수 있도록 해야 합니다.

---

## 8. 정리 및 마무리

이 글에서는 Circuit Breaker 패턴의 개념과 동작 원리, 그리고 Spring 기반 마이크로서비스에서 Resilience4j를 사용하여 적용하는 방법을 정리해보았습니다.

**핵심 포인트:**

- Circuit Breaker는 **장애 전파를 막기 위한 필수 패턴**
- CLOSED → OPEN → HALF-OPEN 상태 전환으로 자동 복구 지원
- Hystrix는 유지보수 모드이므로, **Resilience4j 또는 Spring Cloud Circuit Breaker 사용 권장**
- Fallback 전략을 신중하게 설계해야 함
- 모니터링과 알림을 통해 Circuit Breaker 상태를 지속적으로 관찰해야 함

마이크로서비스 아키텍처에서는 데이터베이스 선택(MySQL vs PostgreSQL)뿐만 아니라, **서비스 간 통신의 안정성**도 매우 중요합니다. Circuit Breaker 패턴을 통해 각 서비스의 장애를 격리하고, 전체 시스템의 가용성을 높일 수 있습니다. 🛡️

다음 글에서는 Circuit Breaker와 함께 사용되는 **Retry 패턴과 Bulkhead 패턴**에 대해 정리해보겠습니다.


