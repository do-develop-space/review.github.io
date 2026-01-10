---
layout: post
title: "멱등성 키 기반 동시성 제어: PK 기반 락 vs Named Lock 성능 비교"
date: 2026-01-10
categories: [architecture, database, spring]
tags: [멱등성, 동시성제어, Idempotency, ConcurrencyControl, PK락, NamedLock, MySQL, 분산락, 성능비교]
---

# 멱등성 키 기반 동시성 제어: PK 기반 락 vs Named Lock 성능 비교

이전 글에서 Pod Disruption Budget(PDB)을 통해 Pod의 가용성을 보장하는 방법을 다뤘습니다. 이번 글에서는 **멱등성 키 기반 동시성 제어**를 구현하는 두 가지 방식인 **PK 기반 락(Table-based Lock)**과 **Named Lock**을 비교하고, 각각의 장단점과 사용 시나리오를 정리해보겠습니다.

주문 API에 멱등성을 구현해야 할 때, 사용자의 중복 클릭이나 네트워크 불안정으로 인한 재시도 요청에서 중복 주문을 방지하는 것이 목표입니다. 이때 동시성 제어가 필수적입니다.

---

## 1. 멱등성과 동시성 제어의 관계

### 1.1 멱등성만으로는 부족한 이유

**멱등성 제어만 있는 경우:**

```java
@RestController
public class OrderController {
    
    @PostMapping("/orders")
    public OrderResponse createOrder(@RequestBody OrderRequest request) {
        String idempotencyKey = request.getIdempotencyKey();
        
        // 멱등성 체크
        Optional<Order> existing = orderRepository.findByIdempotencyKey(idempotencyKey);
        if (existing.isPresent()) {
            return existing.get().toResponse(); // 기존 결과 반환
        }
        
        // 새 주문 생성
        Order order = createOrderInternal(request);
        return order.toResponse();
    }
}
```

**문제: Race Condition 발생**

```
요청 A: 멱등키 확인 → 없음 → 주문 생성 시작
요청 B: 멱등키 확인 → 없음 (A가 아직 저장 안 함) → 주문 생성 시작
→ 중복 주문 발생! ❌
```

### 1.2 멱등성 + 동시성 제어 필요

**멱등성 + 동시성 제어:**

```java
@RestController
public class OrderController {
    
    @PostMapping("/orders")
    public OrderResponse createOrder(@RequestBody OrderRequest request) {
        String idempotencyKey = request.getIdempotencyKey();
        String lockKey = "idempotency:" + idempotencyKey;
        
        // 락으로 동시 접근 제어
        return lockManager.executeWithLock(lockKey, () -> {
            // 락 내부에서 멱등키 재확인 (Race Condition 방지)
            Optional<Order> existing = orderRepository.findByIdempotencyKey(idempotencyKey);
            if (existing.isPresent()) {
                return existing.get().toResponse();
            }
            
            // 새 주문 생성
            Order order = createOrderInternal(request);
            return order.toResponse();
        });
    }
}
```

**해결: Race Condition 방지**

```
요청 A: 락 획득 → 멱등키 확인 → 없음 → 주문 생성 → 저장
요청 B: 락 대기 → 락 획득 → 멱등키 확인 → 있음 → 기존 결과 반환
→ 중복 주문 방지! ✅
```

---

## 2. PK 기반 락 (Table-based Lock)

### 2.1 PK 기반 락의 개념

**PK 기반 락:**
- 별도 락 테이블을 만들고 Primary Key 제약조건을 활용
- INSERT 시도로 락 획득 (PK 제약조건 위반 시 실패)
- DELETE로 락 해제

**락 테이블 구조:**

```sql
CREATE TABLE DISTRIBUTED_LOCKS (
    LOCK_KEY varchar(255) PRIMARY KEY
);
```

**동작 원리:**
1. 락 획득: `INSERT INTO DISTRIBUTED_LOCKS (LOCK_KEY) VALUES ('lock-key')`
   - 성공 → 락 획득
   - 실패 (PK 제약조건 위반) → 락 획득 실패
2. 락 해제: `DELETE FROM DISTRIBUTED_LOCKS WHERE LOCK_KEY = 'lock-key'`

### 2.2 PK 기반 락 구현

**락 매니저 구현:**

```java
@Component
@Slf4j
public class DistributedLockManager {
    
    private final DistributedLockRepository lockRepository;
    
    public <T> T executeWithLock(String lockKey, Supplier<T> supplier) {
        if (enterCriticalZone(lockKey)) {
            try {
                return supplier.get();
            } finally {
                exitCriticalZone(lockKey);
            }
        }
        throw new IllegalStateException("락 획득 실패: " + lockKey);
    }
    
    private boolean enterCriticalZone(String lockKey) {
        try {
            DistributedLock lock = DistributedLock.create(lockKey);
            lockRepository.save(lock); // INSERT로 락 획득
            return true;
        } catch (DataIntegrityViolationException e) {
            // PK 제약조건 위반 = 이미 락 존재
            return false;
        } catch (ObjectOptimisticLockingFailureException e) {
            // 동시 DELETE 시도
            return false;
        } catch (Exception e) {
            log.warn("예상치 못한 락 오류", e);
            return false;
        }
    }
    
    private void exitCriticalZone(String lockKey) {
        try {
            lockRepository.deleteByLockKey(lockKey);
        } catch (Exception e) {
            log.error("락 해제 실패: {}", lockKey, e);
        }
    }
}
```

**엔티티:**

```java
@Entity
@Table(name = "DISTRIBUTED_LOCKS")
public class DistributedLock {
    @Id
    private String lockKey;
    
    public static DistributedLock create(String lockKey) {
        DistributedLock lock = new DistributedLock();
        lock.lockKey = lockKey;
        return lock;
    }
}
```

### 2.3 PK 기반 락의 장단점

**장점:**
- ✅ 표준 SQL로 모든 DB에서 동작
- ✅ 구현이 직관적
- ✅ 락 상태를 쿼리로 확인 가능
- ✅ 적은 Connection Pool에서도 안정적 (10-20개)
- ✅ 순차 처리로 예측 가능한 성능

**단점:**
- ❌ INSERT/DELETE 오버헤드
- ❌ 트랜잭션 의존적
- ❌ 다양한 예외 처리 필요 (DataIntegrityViolationException, ObjectOptimisticLockingFailureException 등)

---

## 3. Named Lock

### 3.1 Named Lock의 개념

**Named Lock:**
- MySQL의 `GET_LOCK()`, `RELEASE_LOCK()`, `IS_USED_LOCK()` 함수 사용
- 서버 메모리의 해시테이블에 락 정보 저장
- `pthread_mutex`로 실제 동시성 제어
- Connection 생명주기와 락이 연동

**MySQL 공식 문서:**
> "These locks are saved in a hash table in the server and implemented with pthread_mutex_lock() and pthread_mutex_unlock() for high speed."

**핵심 특징:**
- 서버 메모리의 해시테이블에 락 정보 저장
- `pthread_mutex`로 실제 동시성 제어
- Connection 생명주기와 락이 연동
- Advisory Lock 방식 (협력적 락킹)

### 3.2 Named Lock 구현

**기본 사용법:**

```sql
-- 락 획득 (최대 3초 대기)
SELECT GET_LOCK('my_lock', 3);
-- 1: 성공, 0: 타임아웃, NULL: 에러

-- 락 해제
SELECT RELEASE_LOCK('my_lock');
-- 1: 성공, 0: 없는 락, NULL: 에러

-- 락 상태 확인
SELECT IS_USED_LOCK('my_lock');
-- Connection ID 또는 NULL
```

**락 매니저 구현:**

```java
@Component
@Slf4j
public class NamedLockManager {
    
    private final DataSource dataSource;
    
    public <T> T executeWithLock(String lockKey, Supplier<T> supplier) {
        try (Connection conn = dataSource.getConnection()) {
            return enterCriticalZone(conn, lockKey, supplier);
        } catch (SQLException e) {
            throw new IllegalStateException("락 처리 중 오류", e);
        }
    }
    
    private <T> T enterCriticalZone(Connection conn, String lockKey, Supplier<T> supplier) throws SQLException {
        // 락 획득
        try (PreparedStatement ps = conn.prepareStatement("SELECT GET_LOCK(?, ?)")) {
            ps.setString(1, lockKey);
            ps.setInt(2, 3); // 3초 타임아웃
            ResultSet rs = ps.executeQuery();
            
            if (!rs.next() || rs.getInt(1) != 1) {
                throw new IllegalStateException("락 획득 실패: " + lockKey);
            }
        }
        
        // 비즈니스 로직 실행
        try {
            return supplier.get();
        } finally {
            // 락 해제
            releaseLock(conn, lockKey);
        }
    }
    
    private void releaseLock(Connection conn, String lockKey) throws SQLException {
        try (PreparedStatement ps = conn.prepareStatement("SELECT RELEASE_LOCK(?)")) {
            ps.setString(1, lockKey);
            ps.executeQuery();
        }
    }
}
```

### 3.3 Named Lock의 장단점

**장점:**
- ✅ 빠른 성능 (메모리 기반, 해시테이블 + pthread_mutex)
- ✅ 단순한 예외 처리 (결과값만 확인)
- ✅ 락 상태 확인 가능

**단점:**
- ❌ Connection Pool 의존성 (많은 Pool 필요, 100+)
- ❌ 블로킹 특성 (락 대기 시간 동안 Connection 점유)
- ❌ 확장성 한계 (동시 요청 증가 시 Connection Pool 증설 필요)
- ❌ MySQL 전용 (다른 DB에서는 사용 불가)

---

## 4. 성능 비교

### 4.1 테스트 환경

**테스트 시나리오:**
- 동시 요청: 100건 (극한 테스트)
- Connection Pool: 10개 (적게) | 100개 (많게)
- 멱등키: 모두 동일 (worst case scenario)
- 목표: 중복 처리 없이 1건만 성공

### 4.2 테스트 결과

**PK 기반 락:**

| Connection Pool | 성공률 | 평균 응답시간 | 최대 응답시간 |
|----------------|--------|--------------|--------------|
| 10개 | 100% | 1,500ms | 2,100ms |
| 100개 | 99% | 1,500ms | 2,100ms |

**Named Lock:**

| Connection Pool | 성공률 | 평균 응답시간 | 최대 응답시간 |
|----------------|--------|--------------|--------------|
| 10개 | 1% | 1,500ms | 100,007ms |
| 100개 | 100% | 1,500ms | 1,500ms |

### 4.3 결과 분석

**PK 기반 락:**
- 적은 Connection Pool에서 안정적 (10개에서 100% 성공률)
- 많은 Connection Pool에서도 안정적 (99% 성공률)
- 순차 처리로 예측 가능한 성능
- 최대 응답시간이 안정적 (2,100ms)

**Named Lock:**
- 적은 Connection Pool에서 심각한 문제 (10개에서 1% 성공률)
- 많은 Connection Pool에서 정상 동작 (100개에서 100% 성공률)
- Connection Pool 부족 시 최대 응답시간 급증 (100,007ms)

**핵심 차이점:**
- **PK 기반 락**: Connection Pool 크기에 덜 민감, 안정적
- **Named Lock**: Connection Pool 크기에 매우 민감, 많은 Pool 필요

---

## 5. Connection Pool 최적화 비교

### 5.1 PK 기반 락

**Connection Pool 설정:**

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 10-20  # 최소한으로 설정
```

**특징:**
- 트랜잭션 단위로 빠른 처리
- Connection 점유 시간이 짧음
- 순차적 처리로 안정적 동작
- 적은 Connection Pool로도 충분

### 5.2 Named Lock

**Connection Pool 설정:**

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 100+  # 대량 설정 필요
```

**특징:**
- 락 대기 시간 = Connection 점유 시간
- 동시 요청 100건 → 100개 Connection 필요
- Connection 부족 시 심각한 성능 저하 (1% 성공률)
- 많은 Connection Pool 필요

---

## 6. 예외 처리 비교

### 6.1 PK 기반 락 예외 처리

**다양한 예외 상황 대응:**

```java
private boolean enterCriticalZone(String lockKey) {
    try {
        lockRepository.save(lock);
        return true;
    } catch (DataIntegrityViolationException e) {
        // 정상적인 락 경합
        return false;
    } catch (ObjectOptimisticLockingFailureException e) {
        // 동시 DELETE 시도
        return false;
    } catch (Exception e) {
        log.warn("예상치 못한 락 오류", e);
        return false;
    }
}
```

**복잡도:** 높음 (여러 예외 처리 필요)

### 6.2 Named Lock 예외 처리

**단순한 결과값 확인:**

```java
private void validateLockResult(ResultSet rs) throws SQLException {
    if (!rs.next() || rs.getInt(1) != 1) {
        throw new IllegalStateException("락 획득 실패");
    }
}
```

**복잡도:** 낮음 (결과값만 확인)

---

## 7. 실제 서비스 환경 고려

### 7.1 현실적인 동시성

**실제 멱등키 충돌 시나리오:**
- 네트워크 불안정: 앱 자동 재시도 2-3회
- 사용자 중복 클릭: 빠른 연속 클릭 2-5회
- 모바일 특수상황: WiFi↔셀룰러 전환 시 재시도

**현실적인 동시성:** 2-5건 정도

**100건 테스트는 50배 안전 마진을 확보한 것:**
- 실제 환경에서는 두 방식 모두 충분히 안정적으로 동작
- 다만, 추가적인 고려 포인트가 발생

### 7.2 악의적 공격 시나리오

**PK 기반 락:**
- 이대로 사용해도 악의적인 여러 요청에서 서버 Resource 문제가 없음
- 적은 Connection Pool로도 안정적

**Named Lock:**
- 악의적인 여러 요청에 대비하기 위한 별도의 방지 로직 필요
- Rate Limiting 등 추가 구현 필요

---

## 8. 선택 기준

### 8.1 PK 기반 락이 적합한 경우

**권장 시나리오:**
- Connection Pool을 최소한으로 유지하고 싶을 때
- 안정성과 예측 가능한 성능이 중요할 때
- 외부 의존성을 최소화하고 싶을 때
- 악의적 공격에 대한 방어가 필요할 때

**예시:**
- 주문 시스템
- 결제 시스템
- 예약 시스템

### 8.2 Named Lock이 적합한 경우

**권장 시나리오:**
- Connection Pool을 충분히 확보할 수 있을 때
- 높은 성능이 필요하고 동시성이 낮을 때
- MySQL 전용 환경일 때
- Rate Limiting 등 추가 방어 로직을 구현할 수 있을 때

**예시:**
- 배치 작업
- 내부 시스템
- 동시성이 낮은 작업

### 8.3 하이브리드 접근

**Named Lock + Rate Limiting:**

```java
@Component
public class RequestThrottler {
    public void checkRequestRate(String userId, String apiPath) {
        String key = "rate_limit:" + userId + ":" + apiPath;
        // 1분간 최대 5회 제한
        if (getCurrentCount(key) >= 5) {
            throw new TooManyRequestsException("잠시 후 다시 시도해주세요");
        }
        incrementCounter(key);
    }
}
```

**장점:**
- 악의적 공격 차단
- 정상적인 재시도는 허용 (2-3회)
- Named Lock의 단순함 유지

**단점:**
- 추가 의존성 (Redis 혹은 추가 테이블)
- Rate Limiting 로직 복잡도

---

## 9. Best Practices

### 9.1 PK 기반 락 Best Practices

**1. 지수 백오프 재시도:**

```java
@Aspect
@Component
public class IdempotencyAspect {
    
    @Around("@annotation(Idempotent)")
    public Object checkIdempotency(ProceedingJoinPoint joinPoint) throws InterruptedException {
        String idempotencyKey = extractIdempotencyKey();
        String lockKey = "idempotency:" + idempotencyKey;
        
        // 재시도 로직
        for (int attempt = 1; attempt <= 5; attempt++) {
            try {
                return lockManager.executeWithLock(lockKey, () -> {
                    // 멱등성 처리
                    return processRequest(joinPoint);
                });
            } catch (IllegalStateException e) {
                // 지수 백오프 재시도
                long delay = 50 * attempt + ThreadLocalRandom.current().nextLong(50);
                Thread.sleep(delay);
            }
        }
        throw new IllegalStateException("멱등성 처리 실패");
    }
}
```

**2. 락 타임아웃 설정:**

```java
@Transactional(timeout = 5)  // 5초 타임아웃
public <T> T executeWithLock(String lockKey, Supplier<T> supplier) {
    // ...
}
```

**3. 락 정리 (Cleanup):**

```java
@Scheduled(fixedRate = 60000)  // 1분마다
public void cleanupExpiredLocks() {
    // 오래된 락 정리 (예: 10분 이상)
    lockRepository.deleteByCreatedAtBefore(LocalDateTime.now().minusMinutes(10));
}
```

### 9.2 Named Lock Best Practices

**1. Connection Pool 충분히 확보:**

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 100  # 충분한 Pool
      minimum-idle: 20
```

**2. Rate Limiting 구현:**

```java
@Component
public class RateLimiter {
    private final RedisTemplate<String, String> redisTemplate;
    
    public void checkRate(String key, int maxRequests, Duration window) {
        String count = redisTemplate.opsForValue().get(key);
        if (count != null && Integer.parseInt(count) >= maxRequests) {
            throw new TooManyRequestsException();
        }
        redisTemplate.opsForValue().increment(key);
        redisTemplate.expire(key, window);
    }
}
```

**3. 락 타임아웃 적절히 설정:**

```java
ps.setInt(2, 3); // 3초 타임아웃 (너무 길면 Connection 점유 시간 증가)
```

---

## 10. 종합 비교표

| 구분 | PK 기반 락 | Named Lock |
|------|-----------|------------|
| **Connection Pool** | 적은 Pool (10-20) | 많은 Pool (100+) |
| **성능** | INSERT/DELETE 오버헤드 | 빠름 (메모리 기반) |
| **안정성** | 높음 (Pool 크기에 덜 민감) | 낮음 (Pool 크기에 매우 민감) |
| **예외 처리** | 복잡 (여러 예외 처리) | 단순 (결과값만 확인) |
| **DB 호환성** | 모든 DB | MySQL 전용 |
| **악의적 공격 대응** | 우수 (적은 Pool로도 안정) | 부족 (Rate Limiting 필요) |
| **구현 복잡도** | 중간 | 낮음 |
| **최대 응답시간** | 안정적 (2,100ms) | 불안정 (100,007ms) |
| **권장 시나리오** | 프로덕션 환경, 안정성 우선 | 내부 시스템, 높은 성능 필요 |

---

## 마무리

**핵심 포인트:**

- **멱등성만으로는 Race Condition이 발생하므로, 동시성 제어가 필수입니다.**
- **PK 기반 락은 적은 Connection Pool에서도 안정적이며, 악의적 공격에 강합니다.**
- **Named Lock은 빠르지만 Connection Pool 의존성이 높고, 많은 Pool이 필요합니다.**
- **실제 서비스 환경(2-5건 동시성)에서는 두 방식 모두 충분히 안정적이지만, 극한 상황을 고려하면 PK 기반 락이 더 안전합니다.**

**최종 권장사항:**

- **프로덕션 환경**: PK 기반 락 (안정성 우선)
- **내부 시스템**: Named Lock + Rate Limiting (성능 우선)
- **일반적인 경우**: PK 기반 락 (외부 의존성 최소화, 안정성)

이론적으로는 Named Lock이 빠르지만, 실제 환경에서는 Connection Pool 의존성이라는 치명적 단점이 있습니다. 자신의 환경에서 직접 테스트하고 정량적으로 비교하는 것이 중요합니다.

이번 글에서는 **DB 레벨의 동시성 제어**를 다뤘다면, 다음 글에서는 **Kafka 메시징 시스템에서 메시지의 안정성을 보장하는 acks 옵션**에 대해 정리해보겠습니다. 메시징 시스템에서도 멱등성과 함께 메시지의 내구성(durability)을 보장하는 것이 중요하기 때문입니다. 🚀



