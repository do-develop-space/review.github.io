---
layout: post
title: "분산 락(Distributed Lock): Redis를 활용한 분산 환경 동시성 제어"
date: 2026-01-06
categories: [architecture, redis, spring]
tags: [분산락, DistributedLock, Redis, 동시성제어, Redisson, Lettuce, 마이크로서비스]
---

# 분산 락(Distributed Lock): Redis를 활용한 분산 환경 동시성 제어

이전 글에서 reservation 서비스가 1개의 서버 인스턴스에서만 실행될 때 비관적 락과 낙관적 락을 사용한 동시성 제어를 다뤘습니다. 하지만 **reservation 서비스의 Pod가 여러 개일 때**(또는 여러 서버 인스턴스가 있는 분산 환경)에는 이러한 방식만으로는 부족합니다.

이번 글에서는 **분산 락(Distributed Lock)**을 통해 여러 Pod(서버 인스턴스) 간의 동시성 제어를 구현하는 방법을 정리해보겠습니다. 특히 **Redis**를 활용한 분산 락 구현에 집중하겠습니다.

---

## 1. 분산 락이 필요한 이유

### 1.1 단일 인스턴스 락의 한계

**기존 방식 (JPA 비관적 락):**

```java
@Transactional
public void createReservation(Long experienceId, Long userId, Integer participantCount) {
    // 비관적 락으로 체험 정보 조회
    Experience experience = experienceRepository.findByIdWithLock(experienceId)
        .orElseThrow(() -> new IllegalArgumentException("Experience not found"));
    
    // 잔여 인원 확인 및 예약 생성
    // ...
}
```

**문제점:**
- reservation 서비스가 1개의 Pod에서만 실행될 때만 효과적
- reservation 서비스의 Pod가 여러 개일 때는 각 Pod가 독립적으로 DB 락을 획득할 수 있어 동시성 제어 불가

**중요: 분산 락이 필요한 기준**

분산 락은 **같은 서비스의 Pod가 여러 개일 때**만 필요합니다.

- ✅ **필요한 경우**: reservation 서비스의 Pod가 여러 개일 때
  - 예: reservation Pod 2개, reservation Pod 3개 등
  - 같은 서비스 내에서 동일한 리소스(Experience)에 대해 여러 Pod가 동시 접근

- ❌ **불필요한 경우**: 
  - reservation 서비스 Pod가 1개일 때
  - 다른 서비스들(order, payment 등)이 각각 Pod 1개씩 있어도 상관없음
  - 서로 다른 서비스 간의 동시성 제어는 각 서비스의 독립적인 락으로 처리

**예시:**
```
시나리오 1: 분산 락 불필요 ✅
- reservation 서비스: Pod 1개
- order 서비스: Pod 1개
- payment 서비스: Pod 1개
→ 각 서비스가 독립적으로 동작하므로 JPA 락으로 충분

시나리오 2: 분산 락 필요 ✅
- reservation 서비스: Pod 3개
- order 서비스: Pod 1개
- payment 서비스: Pod 1개
→ reservation 서비스 Pod가 여러 개이므로 분산 락 필요
```

### 1.2 분산 환경에서의 문제

**시나리오: reservation 서비스 Pod가 여러 개일 때**

```
reservation-service Pod 1
    ↓
DB에서 Experience 조회 (비관적 락)
    ↓
잔여 인원 확인: 3명 ✅

reservation-service Pod 2  (동시에 실행)
    ↓
DB에서 Experience 조회 (비관적 락)
    ↓
잔여 인원 확인: 3명 ✅  (Pod 1과 동시에 확인)

Pod 1: 예약 생성 (3명)
Pod 2: 예약 생성 (2명)

결과: 수용 인원 초과! ❌
```

**문제:**
- 각 Pod가 독립적으로 DB 락을 획득 (DB 락은 Pod 간 공유되지 않음)
- Pod 간 동시성 제어 불가
- Race Condition 발생

### 1.3 분산 락의 필요성

**분산 락(Distributed Lock):**
- 여러 Pod 간 공유되는 락 (Redis 등 외부 저장소 사용)
- Redis, etcd 등을 활용
- 모든 Pod가 동일한 락을 경쟁

**해결:**

```
reservation-service Pod 1
    ↓
Redis 분산 락 획득 시도 ✅
    ↓
DB에서 Experience 조회
    ↓
예약 생성
    ↓
분산 락 해제

reservation-service Pod 2
    ↓
Redis 분산 락 획득 시도 ⏳ (대기)
    ↓
(락 획득 후) DB에서 Experience 조회
    ↓
예약 생성
    ↓
분산 락 해제
```

---

## 2. Redis 분산 락 구현

### 2.1 기본 원리

**Redis 분산 락의 핵심:**
- Redis의 `SET` 명령어에 `NX` (Not eXists) 옵션 사용
- 키가 존재하지 않을 때만 설정 (원자적 연산)
- TTL(Time To Live) 설정으로 데드락 방지

**기본 명령어:**

```redis
# 락 획득 (키가 없을 때만 설정)
SET lock:reservation:123 "lock-value" NX EX 30

# 락 해제 (값 확인 후 삭제)
if GET lock:reservation:123 == "lock-value" then
    DEL lock:reservation:123
end
```

### 2.2 Spring Boot + Lettuce 구현

**의존성 추가:**

```gradle
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-redis'
    implementation 'io.lettuce:lettuce-core'
}
```

**Redis 설정:**

```yaml
spring:
  data:
    redis:
      host: localhost
      port: 6379
      password: ${REDIS_PASSWORD:}
```

**분산 락 서비스 구현:**

```java
@Service
@Slf4j
public class DistributedLockService {
    
    private final StringRedisTemplate redisTemplate;
    private static final String LOCK_PREFIX = "lock:";
    private static final long DEFAULT_LOCK_TIMEOUT = 30; // 30초
    
    public boolean tryLock(String lockKey, long timeoutSeconds) {
        String key = LOCK_PREFIX + lockKey;
        String value = UUID.randomUUID().toString();
        
        Boolean result = redisTemplate.opsForValue()
            .setIfAbsent(key, value, Duration.ofSeconds(timeoutSeconds));
        
        return Boolean.TRUE.equals(result);
    }
    
    public void releaseLock(String lockKey, String lockValue) {
        String key = LOCK_PREFIX + lockKey;
        
        // Lua 스크립트로 원자적 삭제 (값 확인 후 삭제)
        String script = 
            "if redis.call('get', KEYS[1]) == ARGV[1] then " +
            "    return redis.call('del', KEYS[1]) " +
            "else " +
            "    return 0 " +
            "end";
        
        redisTemplate.execute(
            new DefaultRedisScript<>(script, Long.class),
            Collections.singletonList(key),
            lockValue
        );
    }
    
    public String tryLockWithValue(String lockKey, long timeoutSeconds) {
        String key = LOCK_PREFIX + lockKey;
        String value = UUID.randomUUID().toString();
        
        Boolean result = redisTemplate.opsForValue()
            .setIfAbsent(key, value, Duration.ofSeconds(timeoutSeconds));
        
        return Boolean.TRUE.equals(result) ? value : null;
    }
}
```

**사용 예시:**

```java
@Service
@Slf4j
public class ReservationService {
    
    private final DistributedLockService lockService;
    private final ExperienceRepository experienceRepository;
    
    public Reservation createReservation(Long experienceId, Long userId, Integer participantCount) {
        String lockKey = "reservation:" + experienceId;
        String lockValue = null;
        
        try {
            // 분산 락 획득 시도
            lockValue = lockService.tryLockWithValue(lockKey, 30);
            if (lockValue == null) {
                throw new LockAcquisitionException("Failed to acquire lock for reservation");
            }
            
            // 비즈니스 로직 실행
            Experience experience = experienceRepository.findById(experienceId)
                .orElseThrow(() -> new IllegalArgumentException("Experience not found"));
            
            int remainingCapacity = experience.getCapacity() - experience.getReservedCount();
            if (remainingCapacity < participantCount) {
                throw new InsufficientCapacityException("Not enough capacity");
            }
            
            // 예약 생성
            Reservation reservation = createReservationInternal(experience, userId, participantCount);
            
            return reservation;
            
        } finally {
            // 분산 락 해제
            if (lockValue != null) {
                lockService.releaseLock(lockKey, lockValue);
            }
        }
    }
}
```

### 2.3 Redisson을 활용한 구현 (권장)

**Redisson이란:**
- Redis 기반 Java 클라이언트
- 분산 락, 분산 컬렉션 등 제공
- 자동 갱신(Auto-renewal) 기능

**의존성 추가:**

```gradle
dependencies {
    implementation 'org.redisson:redisson-spring-boot-starter:3.24.3'
}
```

**Redisson 설정:**

```yaml
spring:
  data:
    redis:
      host: localhost
      port: 6379
```

```java
@Configuration
public class RedissonConfig {
    
    @Bean
    public RedissonClient redissonClient() {
        Config config = new Config();
        config.useSingleServer()
            .setAddress("redis://localhost:6379");
        return Redisson.create(config);
    }
}
```

**Redisson 분산 락 사용:**

```java
@Service
@Slf4j
public class ReservationService {
    
    private final RedissonClient redissonClient;
    private final ExperienceRepository experienceRepository;
    
    public Reservation createReservation(Long experienceId, Long userId, Integer participantCount) {
        String lockKey = "reservation:" + experienceId;
        RLock lock = redissonClient.getLock(lockKey);
        
        try {
            // 락 획득 (최대 10초 대기, 30초 후 자동 해제)
            boolean acquired = lock.tryLock(10, 30, TimeUnit.SECONDS);
            if (!acquired) {
                throw new LockAcquisitionException("Failed to acquire lock for reservation");
            }
            
            // 비즈니스 로직 실행
            Experience experience = experienceRepository.findById(experienceId)
                .orElseThrow(() -> new IllegalArgumentException("Experience not found"));
            
            int remainingCapacity = experience.getCapacity() - experience.getReservedCount();
            if (remainingCapacity < participantCount) {
                throw new InsufficientCapacityException("Not enough capacity");
            }
            
            Reservation reservation = createReservationInternal(experience, userId, participantCount);
            
            return reservation;
            
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new LockAcquisitionException("Interrupted while acquiring lock", e);
        } finally {
            // 락 해제
            if (lock.isHeldByCurrentThread()) {
                lock.unlock();
            }
        }
    }
}
```

**Redisson의 장점:**
- 자동 락 갱신 (Watchdog)
- 공정 락 (Fair Lock) 지원
- 읽기-쓰기 락 (ReadWriteLock) 지원
- 분산된 환경에서 안정적

---

## 3. 분산 락 패턴

### 3.1 기본 분산 락

**패턴:**

```java
RLock lock = redissonClient.getLock("lock-key");

try {
    // 락 획득
    lock.lock();
    
    // 비즈니스 로직
    doSomething();
    
} finally {
    // 락 해제
    lock.unlock();
}
```

### 3.2 타임아웃 분산 락

**락 획득 대기 시간 제한:**

```java
RLock lock = redissonClient.getLock("lock-key");

try {
    // 최대 5초 대기, 락 유지 시간 30초
    boolean acquired = lock.tryLock(5, 30, TimeUnit.SECONDS);
    
    if (!acquired) {
        throw new LockTimeoutException("Failed to acquire lock within timeout");
    }
    
    doSomething();
    
} finally {
    if (lock.isHeldByCurrentThread()) {
        lock.unlock();
    }
}
```

### 3.3 공정 락 (Fair Lock)

**선착순 방식:**

```java
// 기본 락: 비공정 (성능 우선)
RLock lock = redissonClient.getLock("lock-key");

// 공정 락: 선착순 (공정성 우선)
RFairLock fairLock = redissonClient.getFairLock("lock-key");

try {
    fairLock.lock();
    doSomething();
} finally {
    fairLock.unlock();
}
```

**차이점:**

| 구분 | 기본 락 | 공정 락 |
|------|--------|---------|
| **순서** | 비공정 (성능 우선) | 선착순 (공정성 우선) |
| **성능** | 빠름 | 상대적으로 느림 |
| **사용 사례** | 일반적인 경우 | 순서가 중요한 경우 |

### 3.4 읽기-쓰기 락 (ReadWriteLock)

**읽기와 쓰기 분리:**

```java
RReadWriteLock readWriteLock = redissonClient.getReadWriteLock("lock-key");
RLock readLock = readWriteLock.readLock();
RLock writeLock = readWriteLock.writeLock();

// 읽기 작업
try {
    readLock.lock();
    // 여러 읽기 작업 동시 실행 가능
    readData();
} finally {
    readLock.unlock();
}

// 쓰기 작업
try {
    writeLock.lock();
    // 쓰기 작업은 단독 실행
    writeData();
} finally {
    writeLock.unlock();
}
```

**특징:**
- 읽기 락: 여러 개 동시 획득 가능
- 쓰기 락: 단독 획득만 가능
- 읽기와 쓰기는 상호 배제

---

## 4. 실전 예시: 예약 시스템에 적용

### 4.1 완전한 구현

**분산 락 서비스:**

```java
@Service
@Slf4j
public class DistributedLockService {
    
    private final RedissonClient redissonClient;
    
    public <T> T executeWithLock(String lockKey, long waitTime, long leaseTime, 
                                 TimeUnit timeUnit, Supplier<T> task) {
        RLock lock = redissonClient.getLock(lockKey);
        
        try {
            boolean acquired = lock.tryLock(waitTime, leaseTime, timeUnit);
            if (!acquired) {
                throw new LockAcquisitionException(
                    String.format("Failed to acquire lock: %s", lockKey)
                );
            }
            
            log.debug("Lock acquired: {}", lockKey);
            return task.get();
            
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new LockAcquisitionException("Interrupted while acquiring lock", e);
        } finally {
            if (lock.isHeldByCurrentThread()) {
                lock.unlock();
                log.debug("Lock released: {}", lockKey);
            }
        }
    }
    
    public void executeWithLock(String lockKey, long waitTime, long leaseTime,
                                TimeUnit timeUnit, Runnable task) {
        executeWithLock(lockKey, waitTime, leaseTime, timeUnit, () -> {
            task.run();
            return null;
        });
    }
}
```

**예약 서비스에 적용:**

```java
@Service
@Slf4j
@Transactional
public class ReservationService {
    
    private final DistributedLockService lockService;
    private final ExperienceRepository experienceRepository;
    private final ReservationRepository reservationRepository;
    
    public Reservation createReservation(Long experienceId, Long userId, Integer participantCount) {
        String lockKey = "reservation:" + experienceId;
        
        return lockService.executeWithLock(
            lockKey,
            10,  // 최대 10초 대기
            30,  // 락 유지 시간 30초
            TimeUnit.SECONDS,
            () -> {
                // 비즈니스 로직
                Experience experience = experienceRepository.findById(experienceId)
                    .orElseThrow(() -> new IllegalArgumentException("Experience not found"));
                
                int remainingCapacity = experience.getCapacity() - experience.getReservedCount();
                if (remainingCapacity < participantCount) {
                    throw new InsufficientCapacityException(
                        String.format("Not enough capacity. Remaining: %d, Requested: %d",
                            remainingCapacity, participantCount)
                    );
                }
                
                Reservation reservation = Reservation.builder()
                    .experience(experience)
                    .userId(userId)
                    .participantCount(participantCount)
                    .status(ReservationStatus.PENDING)
                    .build();
                
                reservationRepository.save(reservation);
                
                experience.setReservedCount(experience.getReservedCount() + participantCount);
                experienceRepository.save(experience);
                
                log.info("Reservation created: reservationId={}, experienceId={}, participantCount={}",
                    reservation.getReservationId(), experienceId, participantCount);
                
                return reservation;
            }
        );
    }
}
```

### 4.2 AOP를 활용한 분산 락

**어노테이션 기반 분산 락:**

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface DistributedLock {
    String key();  // 락 키
    long waitTime() default 10;  // 대기 시간 (초)
    long leaseTime() default 30;  // 락 유지 시간 (초)
}
```

**AOP 구현:**

```java
@Aspect
@Component
@Slf4j
public class DistributedLockAspect {
    
    private final DistributedLockService lockService;
    
    @Around("@annotation(distributedLock)")
    public Object around(ProceedingJoinPoint joinPoint, DistributedLock distributedLock) throws Throwable {
        String lockKey = distributedLock.key();
        long waitTime = distributedLock.waitTime();
        long leaseTime = distributedLock.leaseTime();
        
        return lockService.executeWithLock(
            lockKey,
            waitTime,
            leaseTime,
            TimeUnit.SECONDS,
            () -> {
                try {
                    return joinPoint.proceed();
                } catch (Throwable e) {
                    throw new RuntimeException(e);
                }
            }
        );
    }
}
```

**사용 예시:**

```java
@Service
public class ReservationService {
    
    @DistributedLock(
        key = "reservation:#{#experienceId}",
        waitTime = 10,
        leaseTime = 30
    )
    public Reservation createReservation(Long experienceId, Long userId, Integer participantCount) {
        // 비즈니스 로직만 작성
        // 분산 락은 AOP가 자동 처리
        Experience experience = experienceRepository.findById(experienceId)
            .orElseThrow(() -> new IllegalArgumentException("Experience not found"));
        
        // ...
    }
}
```

---

## 5. 분산 락 주의사항

### 5.1 데드락 방지

**문제:**
- 락을 획득한 후 예외 발생
- 락이 해제되지 않음
- 다른 요청이 영구적으로 대기

**해결:**
- TTL(Time To Live) 설정
- try-finally로 락 해제 보장
- Redisson의 Watchdog 사용

### 5.2 락 갱신 (Watchdog)

**Redisson의 자동 갱신:**

```java
RLock lock = redissonClient.getLock("lock-key");

// Watchdog 활성화 (기본값)
lock.lock();  // 자동으로 락 갱신

// Watchdog 비활성화
lock.lock(30, TimeUnit.SECONDS);  // 30초 후 자동 해제
```

**동작 방식:**
- 락 획득 후 주기적으로 TTL 갱신
- 프로세스가 살아있는 동안 락 유지
- 프로세스 종료 시 자동 해제

### 5.3 락 키 설계

**좋은 락 키:**
- 구체적이고 명확한 키
- 예: `reservation:123`, `order:456:payment`

**나쁜 락 키:**
- 너무 일반적인 키
- 예: `lock`, `reservation` (모든 예약에 대해 락)

**락 키 패턴:**

```java
// 리소스별 락
String lockKey = "reservation:" + experienceId;

// 작업별 락
String lockKey = "order:" + orderId + ":payment";

// 사용자별 락
String lockKey = "user:" + userId + ":profile";
```

### 5.4 성능 고려사항

**락 유지 시간:**
- 너무 짧으면: 작업 완료 전 락 해제 → 동시성 문제
- 너무 길면: 다른 요청 대기 시간 증가

**권장:**
- 작업 예상 시간의 2-3배
- 최대 60초 이내

**락 획득 대기 시간:**
- 너무 짧으면: 락 획득 실패 증가
- 너무 길면: 사용자 대기 시간 증가

**권장:**
- 5-10초
- 비즈니스 요구사항에 따라 조정

---

## 6. 분산 락 vs DB 락 비교

### 6.1 성능 비교

| 구분 | DB 락 (비관적 락) | 분산 락 (Redis) |
|------|------------------|----------------|
| **성능** | 느림 (DB 연결 오버헤드) | 빠름 (메모리 기반) |
| **확장성** | DB 연결 수 제한 | 높음 (Redis 클러스터) |
| **지연 시간** | 높음 (~10ms) | 낮음 (~1ms) |
| **부하** | DB 부하 증가 | Redis 부하 증가 |

### 6.2 사용 시나리오

**DB 락이 적합한 경우:**
- 트랜잭션이 중요한 경우
- 데이터 일관성이 최우선인 경우
- 단일 데이터베이스 환경

**분산 락이 적합한 경우:**
- reservation 서비스 Pod가 여러 개일 때 (또는 여러 서버 인스턴스 환경)
- 높은 성능이 필요한 경우
- 캐시나 외부 서비스 호출 전 락 필요

### 6.3 하이브리드 접근

**DB 락 + 분산 락 조합:**

```java
@Service
public class ReservationService {
    
    public Reservation createReservation(Long experienceId, Long userId, Integer participantCount) {
        String lockKey = "reservation:" + experienceId;
        
        // 1. 분산 락으로 서버 간 동시성 제어
        return lockService.executeWithLock(lockKey, 10, 30, TimeUnit.SECONDS, () -> {
            // 2. DB 락으로 데이터 일관성 보장
            Experience experience = experienceRepository.findByIdWithLock(experienceId)
                .orElseThrow(() -> new IllegalArgumentException("Experience not found"));
            
            // 비즈니스 로직
            // ...
        });
    }
}
```

---

## 7. Redis 클러스터 환경

### 7.1 Redis 클러스터 설정

**Redisson 클러스터 설정:**

```java
@Configuration
public class RedissonConfig {
    
    @Bean
    public RedissonClient redissonClient() {
        Config config = new Config();
        config.useClusterServers()
            .addNodeAddress("redis://node1:6379")
            .addNodeAddress("redis://node2:6379")
            .addNodeAddress("redis://node3:6379");
        return Redisson.create(config);
    }
}
```

**분산 락 동작:**
- Redis 클러스터에서도 동일하게 동작
- Redisson이 클러스터 환경을 자동 처리

### 7.2 Redis Sentinel 환경

**Redisson Sentinel 설정:**

```java
@Configuration
public class RedissonConfig {
    
    @Bean
    public RedissonClient redissonClient() {
        Config config = new Config();
        config.useSentinelServers()
            .setMasterName("mymaster")
            .addSentinelAddress("redis://sentinel1:26379")
            .addSentinelAddress("redis://sentinel2:26379")
            .addSentinelAddress("redis://sentinel3:26379");
        return Redisson.create(config);
    }
}
```

---

## 8. 모니터링 및 관찰 가능성

### 8.1 분산 락 메트릭

**메트릭 수집:**

```java
@Component
@Slf4j
public class DistributedLockMetrics {
    
    private final MeterRegistry meterRegistry;
    
    public void recordLockAcquisition(String lockKey, boolean success, Duration duration) {
        meterRegistry.counter("distributed_lock.acquisition",
            "lock_key", lockKey,
            "success", String.valueOf(success)
        ).increment();
        
        if (success) {
            meterRegistry.timer("distributed_lock.acquisition_time",
                "lock_key", lockKey
            ).record(duration);
        }
    }
    
    public void recordLockHeldTime(String lockKey, Duration duration) {
        meterRegistry.timer("distributed_lock.held_time",
            "lock_key", lockKey
        ).record(duration);
    }
    
    public void recordLockTimeout(String lockKey) {
        meterRegistry.counter("distributed_lock.timeout",
            "lock_key", lockKey
        ).increment();
    }
}
```

### 8.2 로깅

```java
@Service
@Slf4j
public class DistributedLockService {
    
    public <T> T executeWithLock(String lockKey, long waitTime, long leaseTime,
                                 TimeUnit timeUnit, Supplier<T> task) {
        long startTime = System.currentTimeMillis();
        RLock lock = redissonClient.getLock(lockKey);
        
        MDC.put("lockKey", lockKey);
        
        try {
            boolean acquired = lock.tryLock(waitTime, leaseTime, timeUnit);
            if (!acquired) {
                log.warn("Failed to acquire lock: {}", lockKey);
                throw new LockAcquisitionException("Failed to acquire lock: " + lockKey);
            }
            
            long acquisitionTime = System.currentTimeMillis() - startTime;
            log.debug("Lock acquired: {} (took {}ms)", lockKey, acquisitionTime);
            
            T result = task.get();
            
            long totalTime = System.currentTimeMillis() - startTime;
            log.debug("Lock released: {} (held for {}ms)", lockKey, totalTime);
            
            return result;
            
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            log.error("Interrupted while acquiring lock: {}", lockKey, e);
            throw new LockAcquisitionException("Interrupted while acquiring lock", e);
        } finally {
            if (lock.isHeldByCurrentThread()) {
                lock.unlock();
            }
            MDC.clear();
        }
    }
}
```

---

## 9. Best Practices

### 9.1 락 키 설계

**권장 패턴:**
- 리소스 ID 포함: `reservation:123`
- 작업 타입 포함: `order:456:payment`
- 네임스페이스 포함: `production:reservation:123`

**피해야 할 패턴:**
- 너무 일반적인 키: `lock`, `reservation`
- 사용자 입력 직접 사용: `reservation:${userInput}` (보안 위험)

### 9.2 락 유지 시간 설정

**권장:**
- 작업 예상 시간의 2-3배
- 최소 10초, 최대 60초
- 모니터링을 통해 실제 작업 시간 확인 후 조정

### 9.3 예외 처리

**락 획득 실패 시:**

```java
try {
    return lockService.executeWithLock(lockKey, 10, 30, TimeUnit.SECONDS, () -> {
        // 비즈니스 로직
    });
} catch (LockAcquisitionException e) {
    // 락 획득 실패 처리
    log.warn("Failed to acquire lock: {}", lockKey);
    throw new BusinessException("The resource is being processed. Please try again later.");
}
```

### 9.4 재시도 전략

**지수 백오프 재시도:**

```java
@Service
public class ReservationService {
    
    @Retryable(value = LockAcquisitionException.class, maxAttempts = 3)
    public Reservation createReservation(Long experienceId, Long userId, Integer participantCount) {
        String lockKey = "reservation:" + experienceId;
        
        return lockService.executeWithLock(lockKey, 10, 30, TimeUnit.SECONDS, () -> {
            // 비즈니스 로직
        });
    }
}
```

---

## 마무리

**핵심 포인트:**

- **분산 락은 reservation 서비스 Pod가 여러 개일 때 Pod 간 동시성 제어를 위한 필수 패턴입니다.**
- **Redis를 활용한 분산 락은 높은 성능과 확장성을 제공합니다.**
- **Redisson을 사용하면 자동 락 갱신, 공정 락 등 고급 기능을 활용할 수 있습니다.**
- **락 키 설계, TTL 설정, 예외 처리를 통해 안정적인 분산 락을 구현할 수 있습니다.**

단일 인스턴스에서의 동시성 제어(비관적 락, 낙관적 락)와 분산 락을 적절히 조합하면, 마이크로서비스 아키텍처에서도 안전하고 효율적인 동시성 제어가 가능합니다.

다음 글에서는 분산 락과 함께 고려해야 하는 **분산 트랜잭션과 Saga 패턴**에 대해 정리해볼 예정입니다. 🚀

