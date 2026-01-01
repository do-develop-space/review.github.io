---
layout: post
title: "체험 예약 시스템의 동시성 제어: 비관적 락과 낙관적 락의 하이브리드 접근"
date: 2026-01-01
categories: [programming, jpa, architecture]
tags: [JPA, 동시성제어, Lock, OptimisticLock, PessimisticLock, 예약시스템, RaceCondition, @Version]
---

# 체험 예약 시스템의 동시성 제어: 비관적 락과 낙관적 락의 하이브리드 접근

이전 글에서 JPA `@Transactional`의 격리 수준과 동시성 문제에 대해 다뤘습니다. 이번 글에서는 **체험 서비스 예약 시스템**을 예시로, 실제 비즈니스 로직에서 발생하는 동시성 문제를 해결하는 방법을 정리해보겠습니다.

예약 시스템에서 가장 흔한 문제는 **"동시에 여러 사용자가 마지막 남은 자리를 예약하려고 할 때"** 발생합니다. 이 문제를 해결하기 위해 **비관적 락(Pessimistic Lock)**과 **낙관적 락(Optimistic Lock)**을 상황에 맞게 조합하는 하이브리드 접근 방식을 살펴보겠습니다.

---

## 1. 문제 상황: Race Condition

### 1.1 예약 시스템의 동시성 문제

**시나리오: 체험 예약 시스템**

```java
@Entity
public class Experience {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String title;
    private Integer capacity;      // 전체 수용 인원
    private Integer reservedCount; // 예약된 인원
    private LocalDateTime startTime;
}

@Entity
public class Reservation {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long reservationId;
    
    @ManyToOne
    private Experience experience;
    
    private Long userId;
    private Integer participantCount; // 예약 인원
    private ReservationStatus status; // PENDING, CONFIRMED, CANCELLED
    private LocalDateTime createdAt;
}
```

**문제가 있는 코드:**

```java
@Service
@Transactional
public class ReservationService {
    
    public Reservation createReservation(Long experienceId, Long userId, Integer participantCount) {
        // 1. 체험 정보 조회
        Experience experience = experienceRepository.findById(experienceId)
            .orElseThrow(() -> new IllegalArgumentException("Experience not found"));
        
        // 2. 잔여 인원 확인
        int remainingCapacity = experience.getCapacity() - experience.getReservedCount();
        if (remainingCapacity < participantCount) {
            throw new InsufficientCapacityException("Not enough capacity");
        }
        
        // 3. 예약 생성
        Reservation reservation = Reservation.builder()
            .experience(experience)
            .userId(userId)
            .participantCount(participantCount)
            .status(ReservationStatus.PENDING)
            .createdAt(LocalDateTime.now())
            .build();
        
        reservationRepository.save(reservation);
        
        // 4. 예약 인원 업데이트
        experience.setReservedCount(experience.getReservedCount() + participantCount);
        experienceRepository.save(experience);
        
        return reservation;
    }
}
```

**Race Condition 발생:**

```
시간    사용자 A                          사용자 B
─────────────────────────────────────────────────────────
T1      조회: capacity=10, reserved=7
        remaining=3 확인 ✅
        
T2                                    조회: capacity=10, reserved=7
                                      remaining=3 확인 ✅
        
T3      예약 생성 (participantCount=3)
        reservedCount = 7 + 3 = 10
        
T4                                    예약 생성 (participantCount=2)
                                      reservedCount = 7 + 2 = 9
                                      
결과    reservedCount = 10 (정상)
        reservedCount = 9 (오류! 실제로는 12명 예약됨)
```

**문제점:**
- 두 사용자가 동시에 잔여 인원을 확인
- 둘 다 예약 가능하다고 판단
- 결과적으로 **수용 인원을 초과한 예약**이 발생

---

## 2. 비관적 락 (Pessimistic Lock)

### 2.1 비관적 락이란?

**비관적 락(Pessimistic Lock):**
- 데이터를 읽는 시점에 **락을 걸어서 다른 트랜잭션의 접근을 차단**
- "다른 사용자가 수정할 가능성이 높다"고 가정
- `SELECT ... FOR UPDATE` 구문 사용

**장점:**
- ✅ Race Condition 완전 방지
- ✅ 데이터 일관성 보장

**단점:**
- ❌ 성능 저하 (락 대기 시간)
- ❌ 데드락 발생 가능성
- ❌ 동시성 감소

### 2.2 예약 생성에 비관적 락 적용

**수정된 코드:**

```java
@Service
@Transactional
public class ReservationService {
    
    public Reservation createReservation(Long experienceId, Long userId, Integer participantCount) {
        // 1. 비관적 락으로 체험 정보 조회 (SELECT FOR UPDATE)
        Experience experience = experienceRepository.findById(experienceId)
            .orElseThrow(() -> new IllegalArgumentException("Experience not found"));
        
        // 비관적 락 적용
        experience = experienceRepository.findByIdWithLock(experienceId)
            .orElseThrow(() -> new IllegalArgumentException("Experience not found"));
        
        // 2. 잔여 인원 확인 (락이 걸려있어서 다른 트랜잭션은 대기)
        int remainingCapacity = experience.getCapacity() - experience.getReservedCount();
        if (remainingCapacity < participantCount) {
            throw new InsufficientCapacityException("Not enough capacity");
        }
        
        // 3. 예약 생성
        Reservation reservation = Reservation.builder()
            .experience(experience)
            .userId(userId)
            .participantCount(participantCount)
            .status(ReservationStatus.PENDING)
            .createdAt(LocalDateTime.now())
            .build();
        
        reservationRepository.save(reservation);
        
        // 4. 예약 인원 업데이트
        experience.setReservedCount(experience.getReservedCount() + participantCount);
        experienceRepository.save(experience);
        
        return reservation;
    }
}
```

**Repository에 비관적 락 메서드 추가:**

```java
@Repository
public interface ExperienceRepository extends JpaRepository<Experience, Long> {
    
    // 비관적 락 (PESSIMISTIC_WRITE)
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT e FROM Experience e WHERE e.id = :id")
    Optional<Experience> findByIdWithLock(@Param("id") Long id);
    
    // 또는 메서드 시그니처에 직접 지정
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    Optional<Experience> findById(Long id);
}
```

**동작 방식:**

```
시간    사용자 A                          사용자 B
─────────────────────────────────────────────────────────
T1      SELECT ... FOR UPDATE
        (락 획득) ✅
        
T2                                    SELECT ... FOR UPDATE
                                      (락 대기...) ⏳
        
T3      조회: capacity=10, reserved=7
        remaining=3 확인 ✅
        예약 생성
        reservedCount = 10
        COMMIT (락 해제)
        
T4                                    (락 획득) ✅
                                      조회: capacity=10, reserved=10
                                      remaining=0 확인 ❌
                                      예외 발생
```

### 2.3 비관적 락 타입

**JPA LockModeType:**

```java
// 1. PESSIMISTIC_READ
// - 공유 락 (Shared Lock)
// - 다른 트랜잭션이 읽기는 가능하지만 쓰기는 불가
@Lock(LockModeType.PESSIMISTIC_READ)
Optional<Experience> findByIdForRead(Long id);

// 2. PESSIMISTIC_WRITE
// - 배타적 락 (Exclusive Lock)
// - 다른 트랜잭션이 읽기/쓰기 모두 불가
@Lock(LockModeType.PESSIMISTIC_WRITE)
Optional<Experience> findByIdForWrite(Long id);

// 3. PESSIMISTIC_FORCE_INCREMENT
// - 비관적 락 + 버전 증가
// - 낙관적 락과 비관적 락을 함께 사용
@Lock(LockModeType.PESSIMISTIC_FORCE_INCREMENT)
Optional<Experience> findByIdWithVersionIncrement(Long id);
```

**실제 SQL:**

```sql
-- PESSIMISTIC_READ
SELECT * FROM experience WHERE id = ? FOR SHARE;

-- PESSIMISTIC_WRITE
SELECT * FROM experience WHERE id = ? FOR UPDATE;
```

### 2.4 타임아웃 설정

**락 대기 시간 제한:**

```java
@Repository
public interface ExperienceRepository extends JpaRepository<Experience, Long> {
    
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @QueryHints({
        @QueryHint(name = "javax.persistence.lock.timeout", value = "5000")  // 5초
    })
    @Query("SELECT e FROM Experience e WHERE e.id = :id")
    Optional<Experience> findByIdWithLock(@Param("id") Long id);
}
```

**또는 application.yml:**

```yaml
spring:
  jpa:
    properties:
      javax.persistence.lock.timeout: 5000  # 5초 (밀리초)
```

**타임아웃 발생 시:**

```java
try {
    Experience experience = experienceRepository.findByIdWithLock(experienceId)
        .orElseThrow(() -> new IllegalArgumentException("Experience not found"));
} catch (LockTimeoutException e) {
    // 락 획득 실패 (다른 트랜잭션이 락을 오래 유지)
    throw new BusinessException("Reservation is being processed by another user. Please try again.");
}
```

---

## 3. 낙관적 락 (Optimistic Lock)

### 3.1 낙관적 락이란?

**낙관적 락(Optimistic Lock):**
- 데이터를 읽는 시점에는 락을 걸지 않음
- 수정 시점에 **버전(Version)을 확인**하여 변경 여부 감지
- "대부분의 경우 충돌이 발생하지 않는다"고 가정

**장점:**
- ✅ 성능 우수 (락 대기 없음)
- ✅ 동시성 높음
- ✅ 데드락 발생 가능성 낮음

**단점:**
- ❌ 충돌 발생 시 재시도 필요
- ❌ 버전 필드 추가 필요

### 3.2 예약 상태 변경에 낙관적 락 적용

**Entity에 버전 필드 추가:**

```java
@Entity
public class Reservation {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long reservationId;
    
    @ManyToOne
    private Experience experience;
    
    private Long userId;
    private Integer participantCount;
    
    @Enumerated(EnumType.STRING)
    private ReservationStatus status;
    
    private LocalDateTime createdAt;
    
    // 낙관적 락을 위한 버전 필드
    @Version
    private Long version;
}
```

**상태 변경 메서드:**

```java
@Service
@Transactional
public class ReservationService {
    
    public void confirmReservation(Long reservationId) {
        // 1. 예약 조회 (락 없음)
        Reservation reservation = reservationRepository.findById(reservationId)
            .orElseThrow(() -> new IllegalArgumentException("Reservation not found"));
        
        // 2. 상태 변경
        if (reservation.getStatus() != ReservationStatus.PENDING) {
            throw new IllegalStateException("Only PENDING reservations can be confirmed");
        }
        
        reservation.setStatus(ReservationStatus.CONFIRMED);
        reservationRepository.save(reservation);
        
        // 3. 저장 시점에 버전 체크
        // - DB의 version과 엔티티의 version이 다르면 OptimisticLockException 발생
    }
    
    public void cancelReservation(Long reservationId) {
        Reservation reservation = reservationRepository.findById(reservationId)
            .orElseThrow(() -> new IllegalArgumentException("Reservation not found"));
        
        if (reservation.getStatus() == ReservationStatus.CANCELLED) {
            throw new IllegalStateException("Reservation is already cancelled");
        }
        
        reservation.setStatus(ReservationStatus.CANCELLED);
        reservationRepository.save(reservation);
    }
}
```

**동작 방식:**

```
시간    사용자 A                          사용자 B
─────────────────────────────────────────────────────────
T1      조회: reservation (version=1)
        
T2                                    조회: reservation (version=1)
        
T3      상태 변경: CONFIRMED
        version=1 → version=2
        UPDATE ... WHERE id=? AND version=1 ✅
        COMMIT
        
T4                                    상태 변경: CANCELLED
                                      version=1 → version=2
                                      UPDATE ... WHERE id=? AND version=1 ❌
                                      (실제 DB version=2)
                                      OptimisticLockException 발생
```

**OptimisticLockException 처리:**

```java
@Service
@Transactional
public class ReservationService {
    
    @Retryable(value = OptimisticLockException.class, maxAttempts = 3)
    public void confirmReservation(Long reservationId) {
        Reservation reservation = reservationRepository.findById(reservationId)
            .orElseThrow(() -> new IllegalArgumentException("Reservation not found"));
        
        if (reservation.getStatus() != ReservationStatus.PENDING) {
            throw new IllegalStateException("Only PENDING reservations can be confirmed");
        }
        
        reservation.setStatus(ReservationStatus.CONFIRMED);
        reservationRepository.save(reservation);
    }
}
```

**또는 수동 재시도:**

```java
public void confirmReservation(Long reservationId) {
    int maxRetries = 3;
    for (int attempt = 0; attempt < maxRetries; attempt++) {
        try {
            Reservation reservation = reservationRepository.findById(reservationId)
                .orElseThrow(() -> new IllegalArgumentException("Reservation not found"));
            
            if (reservation.getStatus() != ReservationStatus.PENDING) {
                throw new IllegalStateException("Only PENDING reservations can be confirmed");
            }
            
            reservation.setStatus(ReservationStatus.CONFIRMED);
            reservationRepository.save(reservation);
            
            return;  // 성공 시 종료
            
        } catch (OptimisticLockException e) {
            if (attempt == maxRetries - 1) {
                throw new BusinessException("Reservation was modified by another user. Please refresh and try again.");
            }
            // 재시도 전 잠시 대기
            try {
                Thread.sleep(100 * (attempt + 1));  // 100ms, 200ms, 300ms
            } catch (InterruptedException ie) {
                Thread.currentThread().interrupt();
                throw new BusinessException("Interrupted during retry", ie);
            }
        }
    }
}
```

### 3.3 @Version 필드의 동작 원리

**DB 스키마:**

```sql
CREATE TABLE reservation (
    reservation_id BIGINT PRIMARY KEY,
    experience_id BIGINT,
    user_id BIGINT,
    participant_count INTEGER,
    status VARCHAR(50),
    created_at TIMESTAMP,
    version BIGINT NOT NULL DEFAULT 0  -- 버전 필드
);
```

**UPDATE 쿼리:**

```sql
-- 첫 번째 수정
UPDATE reservation 
SET status = 'CONFIRMED', version = version + 1 
WHERE reservation_id = ? AND version = 1;

-- 두 번째 수정 (동시에 발생)
UPDATE reservation 
SET status = 'CANCELLED', version = version + 1 
WHERE reservation_id = ? AND version = 1;
-- → 영향받은 행 수: 0 (version이 이미 2로 변경됨)
-- → OptimisticLockException 발생
```

---

## 4. 하이브리드 접근: 상황별 락 전략

### 4.1 권장 구현 전략

**작업별 락 방식 선택:**

| 작업 | 락 방식 | 이유 |
|------|---------|------|
| **예약 생성** | 비관적 락 | capacity 검증 필요 (Race Condition 방지) |
| **상태 변경** | 낙관적 락 | 동시 수정 방지 (성능 우선) |
| **예약 조회** | 락 없음 | 읽기 작업은 락 불필요 |
| **예약 취소** | 낙관적 락 | 상태 변경이므로 낙관적 락 사용 |

### 4.2 완전한 구현 예제

**Entity:**

```java
@Entity
public class Experience {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String title;
    private Integer capacity;
    private Integer reservedCount;
    private LocalDateTime startTime;
}

@Entity
public class Reservation {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long reservationId;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "experience_id")
    private Experience experience;
    
    private Long userId;
    private Integer participantCount;
    
    @Enumerated(EnumType.STRING)
    private ReservationStatus status;
    
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
    
    @Version
    private Long version;  // 낙관적 락용
    
    @PrePersist
    public void prePersist() {
        createdAt = LocalDateTime.now();
        updatedAt = LocalDateTime.now();
    }
    
    @PreUpdate
    public void preUpdate() {
        updatedAt = LocalDateTime.now();
    }
}
```

**Repository:**

```java
@Repository
public interface ExperienceRepository extends JpaRepository<Experience, Long> {
    
    // 비관적 락으로 조회 (예약 생성 시 사용)
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @QueryHints({
        @QueryHint(name = "javax.persistence.lock.timeout", value = "5000")
    })
    @Query("SELECT e FROM Experience e WHERE e.id = :id")
    Optional<Experience> findByIdWithLock(@Param("id") Long id);
}

@Repository
public interface ReservationRepository extends JpaRepository<Reservation, Long> {
    // 낙관적 락은 @Version 필드로 자동 처리
}
```

**Service:**

```java
@Service
@Slf4j
@Transactional
public class ReservationService {
    
    private final ExperienceRepository experienceRepository;
    private final ReservationRepository reservationRepository;
    
    // 1. 예약 생성: 비관적 락 사용
    public Reservation createReservation(Long experienceId, Long userId, Integer participantCount) {
        // 비관적 락으로 체험 정보 조회
        Experience experience = experienceRepository.findByIdWithLock(experienceId)
            .orElseThrow(() -> new IllegalArgumentException("Experience not found"));
        
        // 잔여 인원 확인
        int remainingCapacity = experience.getCapacity() - experience.getReservedCount();
        if (remainingCapacity < participantCount) {
            throw new InsufficientCapacityException(
                String.format("Not enough capacity. Remaining: %d, Requested: %d", 
                    remainingCapacity, participantCount)
            );
        }
        
        // 예약 생성
        Reservation reservation = Reservation.builder()
            .experience(experience)
            .userId(userId)
            .participantCount(participantCount)
            .status(ReservationStatus.PENDING)
            .build();
        
        reservationRepository.save(reservation);
        
        // 예약 인원 업데이트
        experience.setReservedCount(experience.getReservedCount() + participantCount);
        experienceRepository.save(experience);
        
        log.info("Reservation created: reservationId={}, experienceId={}, participantCount={}", 
            reservation.getReservationId(), experienceId, participantCount);
        
        return reservation;
    }
    
    // 2. 예약 확인: 낙관적 락 사용
    @Retryable(value = OptimisticLockException.class, maxAttempts = 3)
    public void confirmReservation(Long reservationId) {
        Reservation reservation = reservationRepository.findById(reservationId)
            .orElseThrow(() -> new IllegalArgumentException("Reservation not found"));
        
        if (reservation.getStatus() != ReservationStatus.PENDING) {
            throw new IllegalStateException(
                String.format("Only PENDING reservations can be confirmed. Current status: %s", 
                    reservation.getStatus())
            );
        }
        
        reservation.setStatus(ReservationStatus.CONFIRMED);
        reservationRepository.save(reservation);
        
        log.info("Reservation confirmed: reservationId={}", reservationId);
    }
    
    // 3. 예약 취소: 낙관적 락 사용
    @Retryable(value = OptimisticLockException.class, maxAttempts = 3)
    public void cancelReservation(Long reservationId) {
        Reservation reservation = reservationRepository.findById(reservationId)
            .orElseThrow(() -> new IllegalArgumentException("Reservation not found"));
        
        if (reservation.getStatus() == ReservationStatus.CANCELLED) {
            throw new IllegalStateException("Reservation is already cancelled");
        }
        
        // 취소 가능한 상태인지 확인
        if (reservation.getStatus() != ReservationStatus.PENDING && 
            reservation.getStatus() != ReservationStatus.CONFIRMED) {
            throw new IllegalStateException(
                String.format("Cannot cancel reservation with status: %s", 
                    reservation.getStatus())
            );
        }
        
        // 예약 취소
        reservation.setStatus(ReservationStatus.CANCELLED);
        reservationRepository.save(reservation);
        
        // 체험의 예약 인원 감소 (비관적 락 필요)
        Experience experience = experienceRepository.findByIdWithLock(
            reservation.getExperience().getId()
        ).orElseThrow(() -> new IllegalArgumentException("Experience not found"));
        
        experience.setReservedCount(
            experience.getReservedCount() - reservation.getParticipantCount()
        );
        experienceRepository.save(experience);
        
        log.info("Reservation cancelled: reservationId={}, participantCount={}", 
            reservationId, reservation.getParticipantCount());
    }
    
    // 4. 예약 조회: 락 없음
    @Transactional(readOnly = true)
    public Reservation getReservation(Long reservationId) {
        return reservationRepository.findById(reservationId)
            .orElseThrow(() -> new IllegalArgumentException("Reservation not found"));
    }
}
```

### 4.3 예외 처리

**커스텀 예외:**

```java
public class InsufficientCapacityException extends BusinessException {
    public InsufficientCapacityException(String message) {
        super(message);
    }
}

public class BusinessException extends RuntimeException {
    public BusinessException(String message) {
        super(message);
    }
    
    public BusinessException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

**Global Exception Handler:**

```java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {
    
    @ExceptionHandler(InsufficientCapacityException.class)
    public ResponseEntity<ErrorResponse> handleInsufficientCapacity(InsufficientCapacityException e) {
        log.warn("Insufficient capacity: {}", e.getMessage());
        return ResponseEntity.status(HttpStatus.CONFLICT)
            .body(new ErrorResponse("INSUFFICIENT_CAPACITY", e.getMessage()));
    }
    
    @ExceptionHandler(OptimisticLockException.class)
    public ResponseEntity<ErrorResponse> handleOptimisticLock(OptimisticLockException e) {
        log.warn("Optimistic lock exception: {}", e.getMessage());
        return ResponseEntity.status(HttpStatus.CONFLICT)
            .body(new ErrorResponse("CONCURRENT_MODIFICATION", 
                "The resource was modified by another user. Please refresh and try again."));
    }
    
    @ExceptionHandler(LockTimeoutException.class)
    public ResponseEntity<ErrorResponse> handleLockTimeout(LockTimeoutException e) {
        log.warn("Lock timeout: {}", e.getMessage());
        return ResponseEntity.status(HttpStatus.REQUEST_TIMEOUT)
            .body(new ErrorResponse("LOCK_TIMEOUT", 
                "The resource is being processed by another user. Please try again later."));
    }
}
```

---

## 5. 성능 고려사항

### 5.1 비관적 락의 성능 영향

**문제점:**
- 락 대기 시간으로 인한 응답 시간 증가
- 동시 처리량 감소

**최적화 방법:**

1. **락 범위 최소화:**
```java
// ❌ 나쁜 예: 전체 트랜잭션 동안 락 유지
@Transactional
public void createReservation(...) {
    Experience experience = findByIdWithLock(...);  // 락 획득
    // ... 긴 처리 시간 ...
    // 락이 오래 유지됨
}

// ✅ 좋은 예: 필요한 최소 시간만 락 유지
@Transactional
public void createReservation(...) {
    // 락 없이 사전 검증
    validateReservation(...);
    
    // 락 획득 → 빠른 처리 → 락 해제
    Experience experience = findByIdWithLock(...);
    updateReservationCount(experience, participantCount);
    // 트랜잭션 커밋 (락 해제)
}
```

2. **락 타임아웃 설정:**
```java
@QueryHints({
    @QueryHint(name = "javax.persistence.lock.timeout", value = "2000")  // 2초
})
```

3. **락 대기 순서 통일 (데드락 방지):**
```java
// 항상 동일한 순서로 락 획득
public void transferReservation(...) {
    Experience exp1 = findByIdWithLock(Math.min(exp1Id, exp2Id));
    Experience exp2 = findByIdWithLock(Math.max(exp1Id, exp2Id));
    // ...
}
```

### 5.2 낙관적 락의 재시도 전략

**재시도 횟수와 간격:**

```java
@Retryable(
    value = OptimisticLockException.class,
    maxAttempts = 3,
    backoff = @Backoff(delay = 100, multiplier = 2)  // 100ms → 200ms → 400ms
)
public void confirmReservation(Long reservationId) {
    // ...
}
```

**재시도 로직 최적화:**

```java
public void confirmReservation(Long reservationId) {
    int maxRetries = 3;
    long baseDelay = 50;  // 50ms
    
    for (int attempt = 0; attempt < maxRetries; attempt++) {
        try {
            Reservation reservation = reservationRepository.findById(reservationId)
                .orElseThrow(() -> new IllegalArgumentException("Reservation not found"));
            
            if (reservation.getStatus() != ReservationStatus.PENDING) {
                throw new IllegalStateException("Only PENDING reservations can be confirmed");
            }
            
            reservation.setStatus(ReservationStatus.CONFIRMED);
            reservationRepository.save(reservation);
            
            return;  // 성공
            
        } catch (OptimisticLockException e) {
            if (attempt == maxRetries - 1) {
                throw new BusinessException("Reservation was modified. Please refresh and try again.");
            }
            
            // Exponential Backoff
            long delay = baseDelay * (1L << attempt);
            try {
                Thread.sleep(delay);
            } catch (InterruptedException ie) {
                Thread.currentThread().interrupt();
                throw new BusinessException("Interrupted during retry", ie);
            }
        }
    }
}
```

---

## 6. 모니터링 및 관찰 가능성

### 6.1 락 관련 메트릭

```java
@Component
@Slf4j
public class LockMetrics {
    
    private final MeterRegistry meterRegistry;
    
    public void recordPessimisticLockAcquisition(String entityType, Duration duration) {
        meterRegistry.timer("lock.pessimistic.acquisition", 
            "entity_type", entityType).record(duration);
    }
    
    public void recordPessimisticLockTimeout(String entityType) {
        meterRegistry.counter("lock.pessimistic.timeout", 
            "entity_type", entityType).increment();
    }
    
    public void recordOptimisticLockConflict(String entityType) {
        meterRegistry.counter("lock.optimistic.conflict", 
            "entity_type", entityType).increment();
    }
    
    public void recordOptimisticLockRetry(String entityType, int retryCount) {
        meterRegistry.counter("lock.optimistic.retry", 
            "entity_type", entityType,
            "retry_count", String.valueOf(retryCount)
        ).increment();
    }
}
```

### 6.2 로깅

```java
@Service
@Slf4j
@Transactional
public class ReservationService {
    
    public Reservation createReservation(Long experienceId, Long userId, Integer participantCount) {
        long startTime = System.currentTimeMillis();
        
        try {
            MDC.put("experienceId", String.valueOf(experienceId));
            MDC.put("userId", String.valueOf(userId));
            
            Experience experience = experienceRepository.findByIdWithLock(experienceId)
                .orElseThrow(() -> new IllegalArgumentException("Experience not found"));
            
            long lockAcquisitionTime = System.currentTimeMillis() - startTime;
            log.debug("Pessimistic lock acquired in {}ms", lockAcquisitionTime);
            
            // ... 예약 생성 로직 ...
            
            long totalTime = System.currentTimeMillis() - startTime;
            log.info("Reservation created successfully in {}ms", totalTime);
            
            return reservation;
            
        } catch (LockTimeoutException e) {
            log.warn("Failed to acquire lock for experienceId: {}", experienceId, e);
            throw e;
        } finally {
            MDC.clear();
        }
    }
}
```

---

## 7. 실전 Best Practices

### 7.1 락 선택 가이드

**비관적 락을 사용해야 하는 경우:**
- ✅ **데이터 무결성이 최우선**인 경우 (예: 재고 차감, 잔액 출금)
- ✅ **충돌이 자주 발생**하는 경우
- ✅ **읽기-수정-쓰기 패턴**에서 수정이 필수인 경우

**낙관적 락을 사용해야 하는 경우:**
- ✅ **충돌이 드물게 발생**하는 경우
- ✅ **성능이 중요**한 경우
- ✅ **읽기 작업이 많고 쓰기 작업이 적은** 경우
- ✅ **재시도가 가능**한 경우

### 7.2 주의사항

1. **비관적 락:**
   - 락 범위를 최소화
   - 타임아웃 설정 필수
   - 데드락 방지를 위한 락 순서 통일

2. **낙관적 락:**
   - 재시도 로직 필수
   - 사용자에게 적절한 에러 메시지 제공
   - 재시도 횟수 제한

3. **하이브리드 접근:**
   - 작업의 특성에 맞는 락 선택
   - 일관된 락 전략 유지
   - 문서화 및 팀 공유

---

## 마무리

**핵심 포인트:**

- **예약 생성은 비관적 락을 사용하여 Race Condition을 완전히 방지**
- **상태 변경은 낙관적 락을 사용하여 성능을 유지하면서 동시 수정 방지**
- **작업의 특성에 맞는 락 전략을 선택하는 것이 중요**
- **락 타임아웃, 재시도 전략, 모니터링을 통해 안정적인 운영 가능**

체험 예약 시스템에서 **비관적 락과 낙관적 락을 하이브리드로 사용**하면, 데이터 일관성을 보장하면서도 성능을 최적화할 수 있습니다. 각 작업의 특성을 분석하여 적절한 락 전략을 선택하는 것이 핵심입니다.

이 글에서 다룬 동시성 제어는 **reservation 서비스가 1개의 서버 인스턴스에서만 실행될 때** 효과적으로 동작하는 방식입니다. JPA의 비관적 락과 낙관적 락은 DB 레벨의 락이지만, 여러 서버 인스턴스(Pod)가 있을 때는 각 서버가 독립적으로 DB 락을 획득할 수 있어 동시성 제어가 불가능합니다. 마이크로서비스 아키텍처나 여러 서버 인스턴스가 있는 분산 환경에서는 **분산 락(Distributed Lock)**이 필요합니다. 분산 락에 대해서는 추후 별도 글로 정리해볼 예정입니다. 🚀


