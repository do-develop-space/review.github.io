---
layout: post
title: "JPA에서 Set/Map 사용 시 equals/hashCode 구현과 영속성 컨텍스트 이슈"
date: 2025-12-16
categories: [programming, jpa]
tags: [JPA, equals, hashCode, 영속성컨텍스트, Set, Map, Lombok, Entity]
---

# JPA에서 Set/Map 사용 시 equals/hashCode 구현과 영속성 컨텍스트 이슈

JPA 엔티티를 `Set`이나 `Map`의 키로 사용할 때, `equals()`와 `hashCode()`를 올바르게 구현하지 않으면 예상치 못한 문제가 발생할 수 있습니다.

특히 **영속성 컨텍스트(Persistence Context)**와 관련된 이슈는 디버깅이 어렵고, 프로덕션 환경에서 심각한 버그를 일으킬 수 있습니다.

이 글에서는 JPA 엔티티에서 `equals()`와 `hashCode()`를 올바르게 구현하는 방법과, 영속성 컨텍스트가 무엇인지, 그리고 왜 문제가 발생하는지 정리해보겠습니다.

---

## 1. 영속성 컨텍스트(Persistence Context)란?

### 영속성 컨텍스트의 개념

**영속성 컨텍스트**는 JPA가 엔티티를 관리하는 논리적인 공간입니다.  
엔티티의 생명주기를 관리하고, 변경 사항을 추적하여 데이터베이스와 동기화합니다.

**주요 특징:**

1. **엔티티 관리**: 영속성 컨텍스트 내에서 관리되는 엔티티는 JPA가 추적
2. **1차 캐시**: 같은 ID를 가진 엔티티는 한 번만 조회되고 캐시됨
3. **변경 감지(Dirty Checking)**: 엔티티의 변경 사항을 자동으로 감지하여 UPDATE 쿼리 생성
4. **지연 로딩(Lazy Loading)**: 연관된 엔티티를 필요할 때만 로드

### 영속성 컨텍스트의 생명주기

```java
// EntityManager를 통해 영속성 컨텍스트 접근
EntityManager em = entityManagerFactory.createEntityManager();

// 트랜잭션 시작
em.getTransaction().begin();

// 엔티티를 영속성 컨텍스트에 저장 (영속 상태)
User user = new User("John");
em.persist(user);  // 영속성 컨텍스트에 추가

// 엔티티 조회 (1차 캐시에서 조회)
User foundUser = em.find(User.class, 1L);  // DB 조회 없이 캐시에서 반환

// 엔티티 수정 (변경 감지)
user.setName("Jane");  // 자동으로 UPDATE 쿼리 생성

// 트랜잭션 커밋
em.getTransaction().commit();

// 영속성 컨텍스트 종료
em.close();
```

### 영속성 컨텍스트와 엔티티 상태

**엔티티의 4가지 상태:**

1. **비영속(Transient)**: 영속성 컨텍스트와 무관한 상태
   ```java
   User user = new User("John");  // 비영속 상태
   ```

2. **영속(Persistent)**: 영속성 컨텍스트에 관리되는 상태
   ```java
   em.persist(user);  // 영속 상태로 전환
   ```

3. **준영속(Detached)**: 영속성 컨텍스트에서 분리된 상태
   ```java
   em.detach(user);  // 준영속 상태로 전환
   // 또는
   em.close();  // 영속성 컨텍스트 종료 시 모든 엔티티가 준영속 상태
   ```

4. **삭제(Removed)**: 삭제 예정 상태
   ```java
   em.remove(user);  // 삭제 상태
   ```

---

## 2. 문제 상황: Set/Map에서의 동일성 비교

### 문제가 발생하는 경우

JPA 엔티티를 `Set`이나 `Map`의 키로 사용할 때, `equals()`와 `hashCode()`가 올바르게 구현되지 않으면 문제가 발생합니다:

```java
@Entity
public class Reservation {
    @Id
    @GeneratedValue
    private Long reservationId;
    
    private String guestName;
    private LocalDateTime checkIn;
    private LocalDateTime checkOut;
    
    // equals()와 hashCode()가 구현되지 않음
}

// 사용하는 쪽
Set<Reservation> reservations = new HashSet<>();
Reservation r1 = reservationRepository.findById(1L).orElseThrow();
Reservation r2 = reservationRepository.findById(1L).orElseThrow();

reservations.add(r1);
reservations.add(r2);  // 같은 엔티티인데 Set에 중복으로 추가될 수 있음!

System.out.println(reservations.size());  // 예상: 1, 실제: 2 (문제!)
```

### 왜 문제가 발생하는가?

**Java의 기본 동작:**
- `Object.equals()`: 참조 비교 (`==`와 동일)
- `Object.hashCode()`: 객체의 메모리 주소 기반 해시 코드

**문제점:**
- 같은 데이터베이스 레코드를 나타내는 엔티티라도, 다른 객체 인스턴스이면 `equals()`가 `false` 반환
- `Set`이나 `Map`에서 중복으로 인식됨

---

## 3. 잘못된 equals/hashCode 구현

### 안티패턴 1: 모든 필드 포함

```java
@Entity
public class Reservation {
    @Id
    @GeneratedValue
    private Long reservationId;
    
    private String guestName;
    private LocalDateTime checkIn;
    private LocalDateTime checkOut;
    
    // ❌ 잘못된 구현: 모든 필드 포함
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        Reservation that = (Reservation) o;
        return Objects.equals(reservationId, that.reservationId) &&
               Objects.equals(guestName, that.guestName) &&
               Objects.equals(checkIn, that.checkIn) &&
               Objects.equals(checkOut, that.checkOut);
    }
    
    @Override
    public int hashCode() {
        return Objects.hash(reservationId, guestName, checkIn, checkOut);
    }
}
```

**문제점:**
- 엔티티가 영속성 컨텍스트에 저장되기 전에는 `reservationId`가 `null`
- `equals()`와 `hashCode()`가 `null`을 포함하여 계산됨
- 영속화 전후로 해시 코드가 변경됨
- `Set`이나 `Map`에서 엔티티를 찾을 수 없게 됨

**실제 문제 시나리오:**
```java
Set<Reservation> reservations = new HashSet<>();

// 1. 비영속 상태의 엔티티 생성
Reservation r1 = new Reservation("John", ...);
reservations.add(r1);  // hashCode: 12345 (guestName, checkIn, checkOut 기반)

// 2. 영속화 (ID 할당)
reservationRepository.save(r1);  // reservationId: 1L 할당

// 3. Set에서 조회 시도
boolean contains = reservations.contains(r1);  // false! (해시 코드가 변경됨)
```

### 안티패턴 2: 가변 필드 포함

```java
@Entity
public class Reservation {
    @Id
    @GeneratedValue
    private Long reservationId;
    
    private String guestName;  // 변경 가능한 필드
    
    // ❌ 잘못된 구현: 가변 필드 포함
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        Reservation that = (Reservation) o;
        return Objects.equals(reservationId, that.reservationId) &&
               Objects.equals(guestName, that.guestName);  // 가변 필드 포함
    }
    
    @Override
    public int hashCode() {
        return Objects.hash(reservationId, guestName);  // 가변 필드 포함
    }
}
```

**문제점:**
- `guestName`이 변경되면 해시 코드도 변경됨
- `Set`이나 `Map`에서 엔티티를 찾을 수 없게 됨

**실제 문제 시나리오:**
```java
Set<Reservation> reservations = new HashSet<>();

Reservation r1 = reservationRepository.findById(1L).orElseThrow();
reservations.add(r1);  // hashCode: 12345

// 필드 변경
r1.setGuestName("Jane");  // hashCode: 67890 (변경됨!)

// Set에서 조회 불가
boolean contains = reservations.contains(r1);  // false!
```

---

## 4. 올바른 equals/hashCode 구현

### 방법 1: ID만 사용 (권장)

**불변 필드인 ID만 사용하여 구현:**

```java
@Entity
public class Reservation {
    @Id
    @GeneratedValue
    private Long reservationId;
    
    private String guestName;
    private LocalDateTime checkIn;
    private LocalDateTime checkOut;
    
    // ✅ 올바른 구현: ID만 사용
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        Reservation that = (Reservation) o;
        return Objects.equals(reservationId, that.reservationId);
    }
    
    @Override
    public int hashCode() {
        return Objects.hash(reservationId);
    }
}
```

**장점:**
- ID는 엔티티의 고유 식별자이므로 변경되지 않음
- 영속화 전후로 해시 코드가 변경되지 않음 (ID가 null이면 null 기반 해시)
- 가변 필드 변경에 영향받지 않음

**주의사항:**
- ID가 `null`인 비영속 엔티티는 서로 다른 객체로 인식됨
- 비영속 엔티티를 `Set`에 추가할 때는 주의 필요

### 방법 2: Lombok의 @EqualsAndHashCode 사용

**Lombok을 사용한 간편한 구현:**

```java
@Entity
@EqualsAndHashCode(callSuper = false, of = "reservationId")
public class Reservation {
    @Id
    @GeneratedValue
    private Long reservationId;
    
    private String guestName;
    private LocalDateTime checkIn;
    private LocalDateTime checkOut;
}
```

**@EqualsAndHashCode 파라미터 설명:**

- `callSuper = false`: 부모 클래스의 `equals()`와 `hashCode()`를 호출하지 않음
  - `true`로 설정하면 부모 클래스의 필드도 포함
  - JPA 엔티티는 보통 `callSuper = false` 사용

- `of = "reservationId"`: `reservationId` 필드만 사용하여 `equals()`와 `hashCode()` 생성
  - 여러 필드를 지정할 수 있음: `of = {"reservationId", "otherField"}`
  - 지정하지 않으면 모든 필드 포함 (비권장)

**생성되는 코드:**
```java
// Lombok이 자동 생성하는 코드
@Override
public boolean equals(Object o) {
    if (this == o) return true;
    if (o == null || getClass() != o.getClass()) return false;
    Reservation that = (Reservation) o;
    return Objects.equals(reservationId, that.reservationId);
}

@Override
public int hashCode() {
    return Objects.hash(reservationId);
}
```

### 방법 3: 비즈니스 키 사용 (ID가 없는 경우)

**ID가 없고 비즈니스 키가 있는 경우:**

```java
@Entity
public class Reservation {
    @Id
    @GeneratedValue
    private Long reservationId;
    
    @NaturalId  // 비즈니스 키
    @Column(unique = true)
    private String confirmationNumber;  // 예약 확인 번호 (불변)
    
    private String guestName;
    
    // ✅ 비즈니스 키 사용
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        Reservation that = (Reservation) o;
        return Objects.equals(confirmationNumber, that.confirmationNumber);
    }
    
    @Override
    public int hashCode() {
        return Objects.hash(confirmationNumber);
    }
}
```

**Lombok 사용:**
```java
@Entity
@EqualsAndHashCode(callSuper = false, of = "confirmationNumber")
public class Reservation {
    @NaturalId
    @Column(unique = true)
    private String confirmationNumber;
    
    // ...
}
```

---

## 5. 영속성 컨텍스트와 equals/hashCode의 관계

### 문제 발생 시나리오

**시나리오 1: 영속화 전후 해시 코드 변경**

```java
Set<Reservation> reservations = new HashSet<>();

// 1. 비영속 엔티티 생성
Reservation r1 = new Reservation("John", ...);
// reservationId = null
// hashCode() = Objects.hash(null) = 0

reservations.add(r1);  // 해시 버킷 0에 저장

// 2. 영속화
reservationRepository.save(r1);  // reservationId = 1L 할당
// hashCode() = Objects.hash(1L) = 32 (변경됨!)

// 3. Set에서 조회 시도
boolean contains = reservations.contains(r1);  
// 해시 버킷 32에서 찾지만, 실제로는 버킷 0에 저장되어 있음
// 결과: false (엔티티를 찾을 수 없음)
```

**시나리오 2: 영속성 컨텍스트에서 조회한 엔티티**

```java
// 같은 트랜잭션 내에서
Reservation r1 = reservationRepository.findById(1L).orElseThrow();
Reservation r2 = reservationRepository.findById(1L).orElseThrow();

// 영속성 컨텍스트의 1차 캐시 덕분에 같은 인스턴스 반환
System.out.println(r1 == r2);  // true (같은 인스턴스)

Set<Reservation> reservations = new HashSet<>();
reservations.add(r1);
reservations.add(r2);  // 같은 인스턴스이므로 중복 추가되지 않음
System.out.println(reservations.size());  // 1
```

**시나리오 3: 다른 트랜잭션에서 조회한 엔티티**

```java
// 트랜잭션 1
Reservation r1 = reservationRepository.findById(1L).orElseThrow();
// 영속성 컨텍스트 1에 저장

// 트랜잭션 2 (다른 영속성 컨텍스트)
Reservation r2 = reservationRepository.findById(1L).orElseThrow();
// 영속성 컨텍스트 2에 저장 (다른 인스턴스)

Set<Reservation> reservations = new HashSet<>();
reservations.add(r1);
reservations.add(r2);  // 다른 인스턴스이지만 같은 ID

// equals()가 ID만 비교하므로
System.out.println(r1.equals(r2));  // true
System.out.println(reservations.size());  // 1 (중복 제거됨)
```

### 영속성 컨텍스트와 1차 캐시

**1차 캐시의 동작:**

```java
EntityManager em = entityManagerFactory.createEntityManager();
em.getTransaction().begin();

// 첫 번째 조회: DB에서 조회 후 1차 캐시에 저장
User user1 = em.find(User.class, 1L);
// SQL: SELECT * FROM users WHERE id = 1

// 두 번째 조회: 1차 캐시에서 반환 (DB 조회 없음)
User user2 = em.find(User.class, 1L);
// SQL 실행 없음

// 같은 인스턴스 반환
System.out.println(user1 == user2);  // true

em.getTransaction().commit();
em.close();
```

**다른 영속성 컨텍스트에서 조회:**

```java
// 영속성 컨텍스트 1
EntityManager em1 = entityManagerFactory.createEntityManager();
User user1 = em1.find(User.class, 1L);
em1.close();

// 영속성 컨텍스트 2 (새로운 컨텍스트)
EntityManager em2 = entityManagerFactory.createEntityManager();
User user2 = em2.find(User.class, 1L);
em2.close();

// 다른 인스턴스 (다른 영속성 컨텍스트)
System.out.println(user1 == user2);  // false

// 하지만 equals()가 ID만 비교하므로
System.out.println(user1.equals(user2));  // true
```

---

## 6. 실전 예시

### 예시 1: Set을 사용한 중복 제거

```java
@Service
public class ReservationService {
    private final ReservationRepository reservationRepository;
    
    // ✅ 올바른 사용: ID 기반 equals/hashCode
    public Set<Reservation> getUniqueReservations(List<Long> reservationIds) {
        Set<Reservation> reservations = new HashSet<>();
        
        for (Long id : reservationIds) {
            reservationRepository.findById(id)
                .ifPresent(reservations::add);
        }
        
        // 같은 ID를 가진 예약은 자동으로 중복 제거됨
        return reservations;
    }
}
```

### 예시 2: Map을 사용한 엔티티 캐싱

```java
@Service
public class ReservationService {
    private final ReservationRepository reservationRepository;
    
    // ✅ 올바른 사용: ID를 키로 사용
    public Map<Long, Reservation> getReservationMap(List<Long> reservationIds) {
        Map<Long, Reservation> reservationMap = new HashMap<>();
        
        for (Long id : reservationIds) {
            reservationRepository.findById(id)
                .ifPresent(reservation -> reservationMap.put(id, reservation));
        }
        
        return reservationMap;
    }
}
```

### 예시 3: @OneToMany에서 Set 사용

```java
@Entity
public class Hotel {
    @Id
    @GeneratedValue
    private Long hotelId;
    
    @OneToMany(mappedBy = "hotel", cascade = CascadeType.ALL)
    private Set<Reservation> reservations = new HashSet<>();
    
    public void addReservation(Reservation reservation) {
        reservations.add(reservation);
        reservation.setHotel(this);
    }
}

@Entity
@EqualsAndHashCode(callSuper = false, of = "reservationId")
public class Reservation {
    @Id
    @GeneratedValue
    private Long reservationId;
    
    @ManyToOne
    @JoinColumn(name = "hotel_id")
    private Hotel hotel;
    
    // equals/hashCode가 ID만 사용하므로 Set에서 중복 제거 정상 동작
}
```

---

## 7. 주의사항 및 Best Practice

### DO (해야 할 것)

1. **ID만 사용하여 equals/hashCode 구현**
   ```java
   @EqualsAndHashCode(callSuper = false, of = "reservationId")
   ```

2. **불변 필드만 사용** (ID가 없는 경우)
   ```java
   @EqualsAndHashCode(callSuper = false, of = "confirmationNumber")
   ```

3. **@NaturalId 활용** (비즈니스 키가 있는 경우)
   ```java
   @NaturalId
   @Column(unique = true)
   private String confirmationNumber;
   ```

### DON'T (하지 말아야 할 것)

1. **모든 필드 포함 금지**
   ```java
   // ❌
   @EqualsAndHashCode  // 모든 필드 포함
   ```

2. **가변 필드 포함 금지**
   ```java
   // ❌
   @EqualsAndHashCode(of = {"reservationId", "guestName"})  // guestName은 변경 가능
   ```

3. **연관 관계 필드 포함 금지**
   ```java
   // ❌
   @EqualsAndHashCode(of = {"reservationId", "hotel"})  // 연관 관계는 변경 가능
   ```

4. **@GeneratedValue가 아닌 필드 사용 주의**
   ```java
   // ❌ ID가 아닌 필드를 사용 (변경 가능)
   @EqualsAndHashCode(of = "guestName")
   ```

### Lombok 사용 시 주의사항

```java
// ✅ 권장: ID만 사용
@EqualsAndHashCode(callSuper = false, of = "reservationId")

// ✅ 권장: 불변 비즈니스 키 사용
@EqualsAndHashCode(callSuper = false, of = "confirmationNumber")

// ❌ 비권장: 모든 필드 포함
@EqualsAndHashCode  // 모든 필드 포함 (가변 필드 포함 시 문제)

// ❌ 비권장: 가변 필드 포함
@EqualsAndHashCode(of = {"reservationId", "guestName"})  // guestName은 변경 가능
```

---

## 8. 디버깅 팁

### 문제 진단

**증상:**
- `Set`이나 `Map`에서 엔티티를 찾을 수 없음
- 중복 제거가 제대로 동작하지 않음
- 영속화 후 `Set`에서 엔티티가 사라짐

**확인 사항:**
```java
// 1. equals/hashCode 구현 확인
Reservation r1 = new Reservation(...);
int hashCode1 = r1.hashCode();

reservationRepository.save(r1);
int hashCode2 = r1.hashCode();

System.out.println("Before: " + hashCode1);
System.out.println("After: " + hashCode2);
// 해시 코드가 변경되면 문제!

// 2. 필드 변경 후 해시 코드 확인
r1.setGuestName("New Name");
int hashCode3 = r1.hashCode();
System.out.println("After change: " + hashCode3);
// 해시 코드가 변경되면 문제!
```

### 해결 방법

1. **ID만 사용하도록 수정**
   ```java
   @EqualsAndHashCode(callSuper = false, of = "reservationId")
   ```

2. **불변 필드만 사용** (ID가 없는 경우)
   ```java
   @EqualsAndHashCode(callSuper = false, of = "confirmationNumber")
   ```

3. **수동으로 equals/hashCode 구현**
   ```java
   @Override
   public boolean equals(Object o) {
       if (this == o) return true;
       if (o == null || getClass() != o.getClass()) return false;
       Reservation that = (Reservation) o;
       return Objects.equals(reservationId, that.reservationId);
   }
   
   @Override
   public int hashCode() {
       return Objects.hash(reservationId);
   }
   ```

---

## 마무리

**핵심 포인트:**

- **영속성 컨텍스트**: JPA가 엔티티를 관리하는 논리적 공간, 1차 캐시와 변경 감지 기능 제공
- **equals/hashCode 구현 원칙**: ID만 사용하거나 불변 비즈니스 키만 사용
- **문제 발생 원인**: 모든 필드 포함 시 영속화 전후 해시 코드 변경, 가변 필드 포함 시 필드 변경 후 해시 코드 변경
- **해결 방법**: `@EqualsAndHashCode(callSuper = false, of = "reservationId")` 사용

JPA 엔티티에서 `equals()`와 `hashCode()`를 올바르게 구현하는 것은 `Set`이나 `Map`을 사용할 때 필수적입니다. 특히 **영속성 컨텍스트의 특성**을 이해하고, **ID나 불변 필드만 사용**하여 구현해야 예상치 못한 버그를 방지할 수 있습니다. 🎯

다음 글에서는 JPA의 **N+1 문제**와 해결 방법에 대해 정리해보겠습니다.

