---
layout: post
title: "Retry와 Bulkhead 패턴: Circuit Breaker와 함께 사용하는 장애 대응 전략"
date: 2025-12-11
categories: [microservices]
tags: [Retry Pattern, Bulkhead Pattern, Resilience4j, Circuit Breaker, 장애격리, 마이크로서비스]
---

# Retry와 Bulkhead 패턴: Circuit Breaker와 함께 사용하는 장애 대응 전략

이전 글에서 **Circuit Breaker 패턴**을 통해 장애 전파를 막는 방법을 살펴보았습니다.  
Circuit Breaker만으로는 부족합니다. 일시적인 네트워크 오류나 서비스의 일시적 부하 상황에서는 **재시도(Retry)**가 필요하고, 리소스 격리를 위해서는 **Bulkhead 패턴**이 필요합니다.

이 글에서는 Circuit Breaker와 함께 사용되는 **Retry 패턴**과 **Bulkhead 패턴**의 개념과 실전 적용 방법을 정리해보겠습니다.

---

## 1. Retry 패턴이란?

### 일시적 오류와 영구적 오류

외부 서비스 호출 시 발생하는 오류는 크게 두 가지로 나눌 수 있습니다:

- **일시적 오류(Transient Failure)**
  - 네트워크 일시적 불안정
  - 서비스의 일시적 과부하
  - 타임아웃
  - **재시도하면 성공할 가능성이 높음**

- **영구적 오류(Permanent Failure)**
  - 잘못된 요청(400 Bad Request)
  - 인증 실패(401 Unauthorized)
  - 권한 없음(403 Forbidden)
  - 리소스 없음(404 Not Found)
  - **재시도해도 성공하지 않음**

Retry 패턴은 **일시적 오류에 대해서만 재시도**를 수행하여, 서비스의 안정성과 가용성을 높입니다.

---

## 2. Retry 전략 설계

### 재시도 횟수와 간격

효과적인 Retry 전략을 수립하기 위해 고려해야 할 요소들:

#### 1. 고정 간격 재시도 (Fixed Interval)

```yaml
resilience4j:
  retry:
    instances:
      orderService:
        maxAttempts: 3
        waitDuration: 1s
```

- 모든 재시도가 동일한 간격으로 수행됨
- 구현이 간단하지만, 서비스가 복구 중일 때 동시에 많은 요청이 몰릴 수 있음

#### 2. 지수 백오프 (Exponential Backoff)

```yaml
resilience4j:
  retry:
    instances:
      orderService:
        maxAttempts: 5
        waitDuration: 500ms
        enableExponentialBackoff: true
        exponentialBackoffMultiplier: 2
        exponentialMaxWaitDuration: 10s
```

- 재시도 간격이 점진적으로 증가 (500ms → 1s → 2s → 4s → 8s)
- 서비스 복구 시간을 고려하여 점진적으로 부하를 증가시킴
- **권장되는 방식**

#### 3. 랜덤 백오프 (Random Backoff)

```yaml
resilience4j:
  retry:
    instances:
      orderService:
        maxAttempts: 4
        waitDuration: 1s
        enableRandomizedWait: true
        randomizedWaitFactor: 0.5
```

- 재시도 간격에 랜덤 요소를 추가
- 여러 클라이언트가 동시에 재시도할 때 **Thundering Herd 문제**를 완화

### 재시도할 예외와 재시도하지 않을 예외

모든 예외에 대해 재시도하면 안 됩니다. **재시도 가능한 예외만 선별**해야 합니다.

```yaml
resilience4j:
  retry:
    instances:
      orderService:
        maxAttempts: 3
        waitDuration: 1s
        retryExceptions:
          - org.springframework.web.client.HttpServerErrorException
          - java.util.concurrent.TimeoutException
          - java.io.IOException
        ignoreExceptions:
          - org.springframework.web.client.HttpClientErrorException
```

- `retryExceptions`: 재시도할 예외 (5xx 서버 오류, 타임아웃, 네트워크 오류)
- `ignoreExceptions`: 재시도하지 않을 예외 (4xx 클라이언트 오류)

---

## 3. Resilience4j Retry 실전 예제

### 의존성 추가

```gradle
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.cloud:spring-cloud-starter-circuitbreaker-resilience4j'
    implementation 'io.github.resilience4j:resilience4j-spring-boot3'
    implementation 'io.github.resilience4j:resilience4j-retry'
}
```

### 설정 파일 (application.yaml)

```yaml
resilience4j:
  retry:
    instances:
      orderService:
        maxAttempts: 3
        waitDuration: 1s
        enableExponentialBackoff: true
        exponentialBackoffMultiplier: 2
        exponentialMaxWaitDuration: 10s
        retryExceptions:
          - org.springframework.web.client.HttpServerErrorException
          - java.util.concurrent.TimeoutException
          - java.io.IOException
        ignoreExceptions:
          - org.springframework.web.client.HttpClientErrorException
```

### 어노테이션 기반 사용

```java
@Service
public class OrderService {
    
    private final RestTemplate restTemplate;
    private final Retry retry;
    
    public OrderService(RestTemplate restTemplate, 
                       RetryRegistry retryRegistry) {
        this.restTemplate = restTemplate;
        this.retry = retryRegistry.retry("orderService");
    }
    
    @Retry(name = "orderService")
    public String getOrderInfo(String orderId) {
        return restTemplate.getForObject(
            "http://order-service/api/orders/" + orderId, 
            String.class
        );
    }
}
```

### Circuit Breaker와 Retry 조합

Circuit Breaker와 Retry를 함께 사용할 때는 **순서가 중요**합니다:

```java
@Service
public class OrderService {
    
    @CircuitBreaker(name = "orderService", fallbackMethod = "getOrderInfoFallback")
    @Retry(name = "orderService")
    @TimeLimiter(name = "orderService")
    public CompletableFuture<String> getOrderInfo(String orderId) {
        return CompletableFuture.supplyAsync(() -> {
            return restTemplate.getForObject(
                "http://order-service/api/orders/" + orderId, 
                String.class
            );
        });
    }
    
    public CompletableFuture<String> getOrderInfoFallback(String orderId, Exception ex) {
        return CompletableFuture.completedFuture(
            "주문 정보를 가져올 수 없습니다. 잠시 후 다시 시도해주세요."
        );
    }
}
```

**실행 순서:**
1. **TimeLimiter**: 타임아웃 체크
2. **Retry**: 일시적 오류에 대해 재시도
3. **Circuit Breaker**: 재시도 후에도 실패하면 Circuit Breaker가 열리고 Fallback 호출

---

## 4. Bulkhead 패턴이란?

### 선박의 격벽(Bulkhead) 개념

선박에는 **격벽(Bulkhead)**이 있습니다. 한 구역에 구멍이 나도 다른 구역으로 물이 넘어가지 않도록 막는 역할을 합니다.

마이크로서비스 아키텍처에서도 동일한 원리가 적용됩니다:

- **리소스 격리**: 한 서비스의 장애나 과부하가 다른 서비스의 리소스를 소진시키지 않도록 격리
- **스레드 풀 격리**: 각 외부 서비스 호출에 대해 독립적인 스레드 풀 사용
- **커넥션 풀 격리**: 각 외부 서비스에 대해 독립적인 커넥션 풀 사용

### 왜 Bulkhead가 필요한가?

**문제 시나리오:**

```
서비스 A → 외부 서비스 X (느림/장애)
서비스 A → 외부 서비스 Y (정상)
```

만약 공유 스레드 풀을 사용한다면:
- 서비스 X 호출이 스레드를 모두 점유
- 서비스 Y 호출도 스레드를 사용할 수 없음
- **정상적인 서비스 Y까지 영향을 받음**

Bulkhead 패턴을 적용하면:
- 서비스 X용 스레드 풀과 서비스 Y용 스레드 풀이 분리
- 서비스 X가 장애가 나도 서비스 Y는 정상 동작
- **장애 격리 효과**

---

## 5. Resilience4j Bulkhead 실전 예제

### Thread Pool Bulkhead

각 외부 서비스 호출에 대해 독립적인 스레드 풀을 사용합니다.

#### 설정 파일 (application.yaml)

```yaml
resilience4j:
  thread-pool-bulkhead:
    instances:
      orderService:
        maxThreadPoolSize: 10
        coreThreadPoolSize: 5
        queueCapacity: 20
      paymentService:
        maxThreadPoolSize: 5
        coreThreadPoolSize: 2
        queueCapacity: 10
```

**설정 설명:**
- `maxThreadPoolSize`: 최대 스레드 수
- `coreThreadPoolSize`: 코어 스레드 수
- `queueCapacity`: 대기 큐 크기

#### 서비스 레이어 구현

```java
@Service
public class OrderService {
    
    private final RestTemplate restTemplate;
    private final ThreadPoolBulkhead threadPoolBulkhead;
    
    public OrderService(RestTemplate restTemplate, 
                       ThreadPoolBulkheadRegistry threadPoolBulkheadRegistry) {
        this.restTemplate = restTemplate;
        this.threadPoolBulkhead = threadPoolBulkheadRegistry.bulkhead("orderService");
    }
    
    public CompletableFuture<String> getOrderInfo(String orderId) {
        Supplier<String> orderSupplier = () -> {
            return restTemplate.getForObject(
                "http://order-service/api/orders/" + orderId, 
                String.class
            );
        };
        
        return threadPoolBulkhead.executeSupplier(orderSupplier);
    }
}
```

#### 어노테이션 기반 사용

```java
@Service
public class OrderService {
    
    @Bulkhead(name = "orderService", type = Bulkhead.Type.THREADPOOL)
    public CompletableFuture<String> getOrderInfo(String orderId) {
        return CompletableFuture.supplyAsync(() -> {
            return restTemplate.getForObject(
                "http://order-service/api/orders/" + orderId, 
                String.class
            );
        });
    }
}
```

### Semaphore Bulkhead

스레드 풀 대신 **세마포어(Semaphore)**를 사용하여 동시 호출 수를 제한합니다.

#### 설정 파일

```yaml
resilience4j:
  bulkhead:
    instances:
      orderService:
        maxConcurrentCalls: 10
        maxWaitDuration: 1s
```

**설정 설명:**
- `maxConcurrentCalls`: 최대 동시 호출 수
- `maxWaitDuration`: 대기 시간 (세마포어를 획득할 수 없을 때)

#### 서비스 레이어 구현

```java
@Service
public class OrderService {
    
    @Bulkhead(name = "orderService", type = Bulkhead.Type.SEMAPHORE)
    public String getOrderInfo(String orderId) {
        return restTemplate.getForObject(
            "http://order-service/api/orders/" + orderId, 
            String.class
        );
    }
}
```

**Thread Pool vs Semaphore:**
- **Thread Pool**: 비동기 처리, 각 서비스별로 독립적인 스레드 풀
- **Semaphore**: 동기 처리, 동시 호출 수만 제한 (스레드 풀은 공유)

---

## 6. Circuit Breaker + Retry + Bulkhead 조합

실전에서는 세 가지 패턴을 함께 사용합니다:

```java
@Service
public class OrderService {
    
    @CircuitBreaker(name = "orderService", fallbackMethod = "getOrderInfoFallback")
    @Retry(name = "orderService")
    @Bulkhead(name = "orderService", type = Bulkhead.Type.THREADPOOL)
    @TimeLimiter(name = "orderService")
    public CompletableFuture<String> getOrderInfo(String orderId) {
        return CompletableFuture.supplyAsync(() -> {
            return restTemplate.getForObject(
                "http://order-service/api/orders/" + orderId, 
                String.class
            );
        });
    }
    
    public CompletableFuture<String> getOrderInfoFallback(String orderId, Exception ex) {
        return CompletableFuture.completedFuture(
            "주문 정보를 가져올 수 없습니다. 잠시 후 다시 시도해주세요."
        );
    }
}
```

**실행 순서:**
1. **Bulkhead**: 스레드 풀에서 실행 (리소스 격리)
2. **TimeLimiter**: 타임아웃 체크
3. **Retry**: 일시적 오류에 대해 재시도
4. **Circuit Breaker**: 재시도 후에도 실패하면 Circuit Breaker가 열리고 Fallback 호출

---

## 7. 통합 설정 예제

실전에서 사용할 수 있는 통합 설정 예제:

```yaml
resilience4j:
  # Circuit Breaker 설정
  circuitbreaker:
    instances:
      orderService:
        slidingWindowSize: 10
        minimumNumberOfCalls: 5
        failureRateThreshold: 50
        waitDurationInOpenState: 10s
        permittedNumberOfCallsInHalfOpenState: 3
  
  # Retry 설정
  retry:
    instances:
      orderService:
        maxAttempts: 3
        waitDuration: 500ms
        enableExponentialBackoff: true
        exponentialBackoffMultiplier: 2
        exponentialMaxWaitDuration: 5s
        retryExceptions:
          - org.springframework.web.client.HttpServerErrorException
          - java.util.concurrent.TimeoutException
  
  # TimeLimiter 설정
  timelimiter:
    instances:
      orderService:
        timeoutDuration: 2s
  
  # Thread Pool Bulkhead 설정
  thread-pool-bulkhead:
    instances:
      orderService:
        maxThreadPoolSize: 10
        coreThreadPoolSize: 5
        queueCapacity: 20
```

---

## 8. 주의사항과 베스트 프랙티스

### Retry 패턴 주의사항

1. **멱등성(Idempotency) 보장**
   - 재시도 시 동일한 작업이 여러 번 수행될 수 있음
   - 멱등하지 않은 작업(예: 결제, 주문 생성)에는 신중하게 적용
   - 멱등 키(Idempotency Key) 사용 권장

2. **재시도 횟수와 간격 조절**
   - 너무 많은 재시도 → 서비스 부하 증가
   - 너무 짧은 간격 → 서비스 복구 시간 부족
   - 지수 백오프 사용 권장

3. **재시도 가능한 예외만 선별**
   - 4xx 클라이언트 오류는 재시도하지 않음
   - 5xx 서버 오류, 타임아웃, 네트워크 오류만 재시도

### Bulkhead 패턴 주의사항

1. **리소스 사용량 모니터링**
   - 각 스레드 풀의 사용률을 모니터링
   - 필요에 따라 스레드 풀 크기 조정

2. **서비스별 우선순위 고려**
   - 중요한 서비스는 더 많은 리소스 할당
   - 덜 중요한 서비스는 제한된 리소스로 운영

3. **Thread Pool vs Semaphore 선택**
   - 비동기 처리가 필요하면 Thread Pool
   - 동기 처리에서 동시 호출 수만 제한하려면 Semaphore

---

## 9. 정리 및 마무리

이 글에서는 Circuit Breaker와 함께 사용되는 **Retry 패턴**과 **Bulkhead 패턴**의 개념과 실전 적용 방법을 정리해보았습니다.

**핵심 포인트:**

- **Retry 패턴**: 일시적 오류에 대해 재시도하여 서비스 안정성 향상
  - 지수 백오프 전략 사용 권장
  - 재시도 가능한 예외만 선별
  - 멱등성 보장 필요

- **Bulkhead 패턴**: 리소스 격리를 통해 장애 전파 방지
  - Thread Pool Bulkhead: 비동기 처리, 독립적인 스레드 풀
  - Semaphore Bulkhead: 동기 처리, 동시 호출 수 제한

- **패턴 조합**: Circuit Breaker + Retry + Bulkhead + TimeLimiter를 함께 사용하여 강력한 장애 대응 체계 구축

마이크로서비스 아키텍처에서 **장애 격리와 복원력(Resilience)**은 매우 중요합니다. Circuit Breaker, Retry, Bulkhead 패턴을 적절히 조합하여 사용하면, 일시적인 장애 상황에서도 서비스가 안정적으로 동작할 수 있습니다. 🛡️

서비스 간 통신의 안정성을 확보하는 것과 마찬가지로, **메시지 큐 시스템 자체의 아키텍처 발전**도 중요합니다. 다음 글에서는 Kafka가 ZooKeeper 의존성을 제거하고 KRaft 모드로 발전하게 된 배경과 계기에 대해 정리해보겠습니다.

