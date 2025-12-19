---
layout: post
title: "JPA @Transactional 완전 가이드: 동작 원리와 주의사항"
date: 2025-12-19
categories: [programming, jpa]
tags: [JPA, Transactional, 트랜잭션, Spring, ACID, 격리수준, 전파속성]
---

# JPA @Transactional 완전 가이드: 동작 원리와 주의사항

JPA와 Spring을 사용할 때 `@Transactional`은 가장 중요한 어노테이션 중 하나입니다.  
하지만 단순히 "트랜잭션을 시작하고 커밋한다"는 것 이상으로, **어떻게 동작하는지**, **언제 사용해야 하는지**, **어떤 주의사항이 있는지**를 이해하는 것이 중요합니다.

이 글에서는 `@Transactional`의 동작 원리, 속성 설정, 그리고 실전에서 자주 발생하는 문제들과 해결 방법을 정리해보겠습니다.

---

## 1. 트랜잭션이란?

### 트랜잭션의 개념

**트랜잭션(Transaction)**은 데이터베이스 작업의 논리적 단위입니다.  
여러 작업을 하나의 단위로 묶어서, **모두 성공하거나 모두 실패**하도록 보장합니다.

**ACID 속성:**
- **Atomicity (원자성)**: 모든 작업이 성공하거나 모두 롤백
- **Consistency (일관성)**: 데이터의 일관성 유지
- **Isolation (격리성)**: 동시 실행되는 트랜잭션 간 격리
- **Durability (지속성)**: 커밋된 데이터는 영구 저장

### 트랜잭션의 필요성

**문제 상황:**
```java
// 계좌 이체 예시
public void transfer(Long fromId, Long toId, BigDecimal amount) {
    // 1. 출금
    Account fromAccount = accountRepository.findById(fromId).orElseThrow();
    fromAccount.withdraw(amount);
    accountRepository.save(fromAccount);
    
    // 2. 입금
    Account toAccount = accountRepository.findById(toId).orElseThrow();
    toAccount.deposit(amount);
    accountRepository.save(toAccount);
    
    // 만약 입금 중 오류 발생 시?
    // 출금은 이미 완료되었는데 입금이 실패하면?
}
```

**해결: 트랜잭션 사용**
```java
@Transactional
public void transfer(Long fromId, Long toId, BigDecimal amount) {
    // 모든 작업이 하나의 트랜잭션으로 묶임
    // 하나라도 실패하면 모두 롤백
    Account fromAccount = accountRepository.findById(fromId).orElseThrow();
    fromAccount.withdraw(amount);
    accountRepository.save(fromAccount);
    
    Account toAccount = accountRepository.findById(toId).orElseThrow();
    toAccount.deposit(amount);
    accountRepository.save(toAccount);
}
```

---

## 2. @Transactional의 동작 원리

### Spring의 트랜잭션 관리

**Spring의 트랜잭션 관리 방식:**
1. **트랜잭션 프록시 생성**: `@Transactional`이 적용된 메서드를 프록시로 감쌈
2. **트랜잭션 시작**: 메서드 호출 전 트랜잭션 시작
3. **메서드 실행**: 실제 메서드 실행
4. **커밋/롤백**: 정상 종료 시 커밋, 예외 발생 시 롤백

**동작 흐름:**
```
클라이언트
  ↓
프록시 객체 (@Transactional)
  ↓
트랜잭션 시작 (begin)
  ↓
실제 메서드 실행
  ↓
트랜잭션 커밋/롤백 (commit/rollback)
```

### 프록시 기반 동작

```java
@Service
public class UserService {
    @Transactional
    public void createUser(String name) {
        User user = new User(name);
        userRepository.save(user);
    }
}

// Spring이 생성하는 프록시 (개념적)
public class UserServiceProxy extends UserService {
    private UserService target;
    private TransactionManager transactionManager;
    
    @Override
    public void createUser(String name) {
        TransactionStatus status = transactionManager.getTransaction(new DefaultTransactionDefinition());
        try {
            target.createUser(name);  // 실제 메서드 호출
            transactionManager.commit(status);
        } catch (Exception e) {
            transactionManager.rollback(status);
            throw e;
        }
    }
}
```

### 내부 메서드 호출 문제

**문제 상황:**
```java
@Service
public class UserService {
    public void createUser(String name) {
        // 내부 메서드 호출
        saveUser(name);  // @Transactional이 적용되지 않음!
    }
    
    @Transactional
    private void saveUser(String name) {
        User user = new User(name);
        userRepository.save(user);
    }
}
```

**원인:**
- 프록시는 **외부에서 호출**될 때만 동작
- 같은 클래스 내부에서 호출하면 프록시를 거치지 않음

**해결 방법:**
```java
@Service
public class UserService {
    @Autowired
    private UserService self;  // 자기 자신을 주입
    
    public void createUser(String name) {
        self.saveUser(name);  // 프록시를 통해 호출
    }
    
    @Transactional
    public void saveUser(String name) {
        User user = new User(name);
        userRepository.save(user);
    }
}
```

또는

```java
@Service
public class UserService {
    public void createUser(String name) {
        // @Transactional을 외부 메서드에 적용
        saveUser(name);
    }
    
    @Transactional
    public void saveUser(String name) {
        User user = new User(name);
        userRepository.save(user);
    }
}
```

---

## 3. @Transactional 속성

### isolation (격리 수준)

**격리 수준이란?**
- 동시에 실행되는 트랜잭션 간의 격리 정도
- 격리 수준이 낮을수록 성능은 좋지만, 데이터 일관성 문제 발생 가능

**격리 수준 종류:**

1. **READ_UNCOMMITTED (레벨 0)**
   ```java
   @Transactional(isolation = Isolation.READ_UNCOMMITTED)
   ```
   - 다른 트랜잭션의 커밋되지 않은 데이터 읽기 가능
   - Dirty Read 발생 가능
   - 가장 빠름, 하지만 데이터 일관성 보장 안 됨

2. **READ_COMMITTED (레벨 1)** - 기본값
   ```java
   @Transactional(isolation = Isolation.READ_COMMITTED)
   ```
   - 커밋된 데이터만 읽기
   - Dirty Read 방지
   - Non-Repeatable Read 발생 가능

3. **REPEATABLE_READ (레벨 2)**
   ```java
   @Transactional(isolation = Isolation.REPEATABLE_READ)
   ```
   - 같은 쿼리를 여러 번 실행해도 같은 결과
   - Non-Repeatable Read 방지
   - Phantom Read 발생 가능

4. **SERIALIZABLE (레벨 3)**
   ```java
   @Transactional(isolation = Isolation.SERIALIZABLE)
   ```
   - 가장 엄격한 격리 수준
   - 모든 동시성 문제 방지
   - 가장 느림, 데드락 발생 가능

**격리 수준별 문제:**

| 격리 수준 | Dirty Read | Non-Repeatable Read | Phantom Read |
|-----------|------------|---------------------|--------------|
| READ_UNCOMMITTED | ✅ 발생 | ✅ 발생 | ✅ 발생 |
| READ_COMMITTED | ❌ 방지 | ✅ 발생 | ✅ 발생 |
| REPEATABLE_READ | ❌ 방지 | ❌ 방지 | ✅ 발생 |
| SERIALIZABLE | ❌ 방지 | ❌ 방지 | ❌ 방지 |

### propagation (전파 속성)

**전파 속성이란?**
- 트랜잭션이 이미 존재할 때 새로운 트랜잭션을 어떻게 처리할지 결정

**전파 속성 종류:**

1. **REQUIRED (기본값)**
   ```java
   @Transactional(propagation = Propagation.REQUIRED)
   ```
   - 기존 트랜잭션이 있으면 참여, 없으면 새로 생성
   - 가장 일반적으로 사용

2. **REQUIRES_NEW**
   ```java
   @Transactional(propagation = Propagation.REQUIRES_NEW)
   ```
   - 항상 새로운 트랜잭션 생성
   - 기존 트랜잭션과 독립적으로 동작
   - 로그 기록 등 독립적인 작업에 사용

3. **SUPPORTS**
   ```java
   @Transactional(propagation = Propagation.SUPPORTS)
   ```
   - 기존 트랜잭션이 있으면 참여, 없으면 트랜잭션 없이 실행

4. **NOT_SUPPORTED**
   ```java
   @Transactional(propagation = Propagation.NOT_SUPPORTED)
   ```
   - 트랜잭션 없이 실행
   - 기존 트랜잭션은 일시 중지

5. **MANDATORY**
   ```java
   @Transactional(propagation = Propagation.MANDATORY)
   ```
   - 기존 트랜잭션이 반드시 있어야 함
   - 없으면 예외 발생

6. **NEVER**
   ```java
   @Transactional(propagation = Propagation.NEVER)
   ```
   - 트랜잭션이 없어야 함
   - 있으면 예외 발생

7. **NESTED**
   ```java
   @Transactional(propagation = Propagation.NESTED)
   ```
   - 중첩 트랜잭션 생성
   - Savepoint를 사용하여 부분 롤백 가능

**실전 예시:**
```java
@Service
public class OrderService {
    @Transactional(propagation = Propagation.REQUIRED)
    public void createOrder(OrderRequest request) {
        Order order = orderRepository.save(new Order(request));
        
        // 로그는 독립적인 트랜잭션으로 처리
        logService.logOrderCreation(order.getId());
        
        // 주문 생성 실패 시 로그는 커밋됨
    }
}

@Service
public class LogService {
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void logOrderCreation(Long orderId) {
        // 항상 새로운 트랜잭션으로 실행
        // OrderService의 트랜잭션과 독립적
        logRepository.save(new Log("Order created: " + orderId));
    }
}
```

### timeout (타임아웃)

**트랜잭션 타임아웃 설정:**
```java
@Transactional(timeout = 10)  // 10초
public void longRunningOperation() {
    // 10초 이상 걸리면 예외 발생
}
```

**기본값:**
- 기본 타임아웃 없음 (무한 대기)
- 데이터베이스 기본 타임아웃 사용

### readOnly (읽기 전용)

**읽기 전용 트랜잭션:**
```java
@Transactional(readOnly = true)
public List<User> findAllUsers() {
    return userRepository.findAll();
}
```

**장점:**
- 읽기 전용이므로 최적화 가능
- 쓰기 작업 시 예외 발생 (데이터 보호)
- 성능 향상 (일부 데이터베이스에서)

**주의사항:**
- `readOnly = true`여도 쓰기 작업이 가능할 수 있음 (JPA 구현에 따라)
- 명시적으로 쓰기를 방지하려면 추가 검증 필요

### rollbackFor / noRollbackFor

**롤백 조건 설정:**
```java
// 특정 예외에서만 롤백
@Transactional(rollbackFor = {IllegalArgumentException.class, NullPointerException.class})
public void processOrder(Order order) {
    // IllegalArgumentException이나 NullPointerException 발생 시에만 롤백
}

// 특정 예외에서 롤백하지 않음
@Transactional(noRollbackFor = {BusinessException.class})
public void processOrder(Order order) {
    // BusinessException 발생 시 롤백하지 않음
}
```

**기본 동작:**
- `RuntimeException`과 `Error`는 롤백
- `Checked Exception`은 롤백하지 않음

---

## 4. @Transactional 사용 위치

### 클래스 레벨 vs 메서드 레벨

**클래스 레벨:**
```java
@Service
@Transactional  // 모든 public 메서드에 적용
public class UserService {
    public void createUser(String name) { }
    public void updateUser(Long id, String name) { }
}
```

**메서드 레벨:**
```java
@Service
public class UserService {
    @Transactional
    public void createUser(String name) { }
    
    @Transactional(readOnly = true)
    public List<User> findAll() { }
    
    public void simpleMethod() { }  // 트랜잭션 없음
}
```

**우선순위:**
- 메서드 레벨이 클래스 레벨보다 우선
- 메서드 레벨에서 더 구체적인 설정 가능

### Repository vs Service

**Repository 레벨:**
```java
@Repository
@Transactional(readOnly = true)  // 기본적으로 읽기 전용
public interface UserRepository extends JpaRepository<User, Long> {
    @Transactional  // 쓰기 작업은 별도 트랜잭션
    @Modifying
    @Query("UPDATE User u SET u.name = :name WHERE u.id = :id")
    void updateName(@Param("id") Long id, @Param("name") String name);
}
```

**Service 레벨 (권장):**
```java
@Service
public class UserService {
    private final UserRepository userRepository;
    
    @Transactional
    public void createUser(String name) {
        User user = new User(name);
        userRepository.save(user);
        // 여러 Repository 호출 가능
        // 비즈니스 로직 포함
    }
}
```

**권장 사항:**
- **Service 레벨에서 @Transactional 사용 권장**
- Repository는 데이터 접근만 담당
- Service에서 비즈니스 로직과 트랜잭션 경계 관리

---

## 5. 실전 예시 및 주의사항

### 예시 1: 트랜잭션 경계

```java
@Service
public class OrderService {
    @Transactional
    public void createOrder(OrderRequest request) {
        // 1. 주문 생성
        Order order = orderRepository.save(new Order(request));
        
        // 2. 재고 차감
        inventoryService.decreaseStock(request.getProductId(), request.getQuantity());
        
        // 3. 결제 처리
        paymentService.processPayment(order.getId(), request.getAmount());
        
        // 모든 작업이 하나의 트랜잭션으로 묶임
        // 하나라도 실패하면 모두 롤백
    }
}
```

### 예시 2: 읽기 전용 트랜잭션

```java
@Service
public class UserService {
    @Transactional(readOnly = true)
    public List<User> findAllUsers() {
        return userRepository.findAll();
    }
    
    @Transactional(readOnly = true)
    public User findUserById(Long id) {
        return userRepository.findById(id).orElseThrow();
    }
}
```

### 예시 3: 예외 처리와 롤백

```java
@Service
public class OrderService {
    @Transactional(rollbackFor = Exception.class)  // 모든 예외에서 롤백
    public void createOrder(OrderRequest request) {
        try {
            Order order = orderRepository.save(new Order(request));
            // ...
        } catch (Exception e) {
            // 예외 발생 시 자동 롤백
            log.error("Order creation failed", e);
            throw e;  // 예외를 다시 던져야 롤백됨
        }
    }
}
```

**주의:**
- 예외를 잡아서 처리하면 롤백되지 않을 수 있음
- 롤백을 원하면 예외를 다시 던져야 함

### 예시 4: LazyInitializationException 방지

**문제 상황:**
```java
@Service
public class UserService {
    public User getUserWithOrders(Long id) {
        User user = userRepository.findById(id).orElseThrow();
        // 트랜잭션이 종료된 후
        return user;  // LazyInitializationException 발생 가능
    }
}
```

**해결 방법:**
```java
@Service
public class UserService {
    @Transactional(readOnly = true)
    public User getUserWithOrders(Long id) {
        User user = userRepository.findById(id).orElseThrow();
        // 트랜잭션이 유지되는 동안
        user.getOrders().size();  // 초기화
        return user;
    }
}
```

또는 Fetch Join 사용:
```java
@Query("SELECT u FROM User u JOIN FETCH u.orders WHERE u.id = :id")
Optional<User> findByIdWithOrders(@Param("id") Long id);
```

---

## 6. 자주 발생하는 문제와 해결

### 문제 1: 내부 메서드 호출

**문제:**
```java
@Service
public class UserService {
    public void createUser(String name) {
        saveUser(name);  // @Transactional 무시됨
    }
    
    @Transactional
    private void saveUser(String name) {
        userRepository.save(new User(name));
    }
}
```

**해결:**
```java
@Service
public class UserService {
    @Autowired
    private UserService self;
    
    public void createUser(String name) {
        self.saveUser(name);  // 프록시를 통해 호출
    }
    
    @Transactional
    public void saveUser(String name) {
        userRepository.save(new User(name));
    }
}
```

### 문제 2: 트랜잭션 범위 밖에서 Lazy Loading

**문제:**
```java
@Service
public class UserService {
    public UserDTO getUserDTO(Long id) {
        User user = userRepository.findById(id).orElseThrow();
        // 트랜잭션 종료 후
        return UserDTO.of(user);  // user.getOrders() 호출 시 LazyInitializationException
    }
}
```

**해결:**
```java
@Service
public class UserService {
    @Transactional(readOnly = true)
    public UserDTO getUserDTO(Long id) {
        User user = userRepository.findById(id).orElseThrow();
        user.getOrders().size();  // 트랜잭션 내에서 초기화
        return UserDTO.of(user);
    }
}
```

### 문제 3: 예외를 잡아서 롤백되지 않음

**문제:**
```java
@Service
public class OrderService {
    @Transactional
    public void createOrder(OrderRequest request) {
        try {
            orderRepository.save(new Order(request));
        } catch (Exception e) {
            log.error("Error", e);
            // 예외를 잡아서 처리하면 롤백되지 않음!
        }
    }
}
```

**해결:**
```java
@Service
public class OrderService {
    @Transactional
    public void createOrder(OrderRequest request) {
        try {
            orderRepository.save(new Order(request));
        } catch (Exception e) {
            log.error("Error", e);
            throw e;  // 예외를 다시 던져야 롤백됨
        }
    }
}
```

### 문제 4: 비동기 메서드에서 @Transactional

**문제:**
```java
@Service
public class UserService {
    @Async
    @Transactional
    public void createUserAsync(String name) {
        // 비동기 메서드는 별도 스레드에서 실행
        // 트랜잭션이 제대로 전파되지 않을 수 있음
        userRepository.save(new User(name));
    }
}
```

**해결:**
```java
@Service
public class UserService {
    @Async
    public void createUserAsync(String name) {
        // 비동기 메서드 내부에서 트랜잭션 시작
        createUser(name);
    }
    
    @Transactional
    public void createUser(String name) {
        userRepository.save(new User(name));
    }
}
```

---

## 7. Best Practice

### DO (해야 할 것)

1. **Service 레벨에서 @Transactional 사용**
   ```java
   @Service
   public class UserService {
       @Transactional
       public void createUser(String name) { }
   }
   ```

2. **읽기 전용 메서드에 readOnly = true 설정**
   ```java
   @Transactional(readOnly = true)
   public List<User> findAll() { }
   ```

3. **트랜잭션 범위를 최소화**
   ```java
   // ❌ 트랜잭션 범위가 너무 넓음
   @Transactional
   public void processLargeData() {
       List<Data> dataList = loadLargeData();  // 트랜잭션 밖에서 처리
       for (Data data : dataList) {
           process(data);
       }
   }
   
   // ✅ 트랜잭션 범위 최소화
   public void processLargeData() {
       List<Data> dataList = loadLargeData();
       for (Data data : dataList) {
           processInTransaction(data);
       }
   }
   
   @Transactional
   private void processInTransaction(Data data) {
       process(data);
   }
   ```

4. **예외 처리 시 롤백 확인**
   ```java
   @Transactional(rollbackFor = Exception.class)
   public void process() {
       try {
           // ...
       } catch (Exception e) {
           throw e;  // 롤백을 위해 예외 다시 던지기
       }
   }
   ```

### DON'T (하지 말아야 할 것)

1. **내부 메서드 호출로 @Transactional 우회 금지**
   ```java
   // ❌
   public void method() {
       privateTransactionalMethod();
   }
   ```

2. **트랜잭션 범위를 너무 넓게 설정 금지**
   ```java
   // ❌
   @Transactional
   public void processLargeData() {
       // 수백만 건 처리
   }
   ```

3. **예외를 잡아서 롤백 방지 금지**
   ```java
   // ❌
   @Transactional
   public void method() {
       try {
           // ...
       } catch (Exception e) {
           // 예외를 잡아서 처리하면 롤백 안 됨
       }
   }
   ```

4. **비동기 메서드에 직접 @Transactional 사용 주의**
   ```java
   // ⚠️ 주의 필요
   @Async
   @Transactional
   public void asyncMethod() { }
   ```

---

## 마무리

**핵심 포인트:**

- **@Transactional 동작 원리**: 프록시 기반으로 트랜잭션 시작/커밋/롤백 처리
- **속성 설정**: isolation, propagation, timeout, readOnly, rollbackFor
- **사용 위치**: Service 레벨에서 사용 권장, 메서드 레벨이 클래스 레벨보다 우선
- **주의사항**: 내부 메서드 호출 문제, LazyInitializationException, 예외 처리

`@Transactional`은 JPA와 Spring에서 가장 중요한 어노테이션 중 하나입니다. **트랜잭션의 동작 원리와 속성을 이해**하고, **실전에서 자주 발생하는 문제들을 인지**하여 올바르게 사용해야 합니다. 특히 **내부 메서드 호출 문제**와 **LazyInitializationException**은 자주 발생하는 문제이므로 주의가 필요합니다. 🎯

트랜잭션 경계를 적절히 나누는 것은 마이크로서비스 간 통신에서도 중요합니다. 다음 글에서는 OpenFeign과 Jackson이 어떻게 연계되어 HTTP 요청/응답을 JSON으로 직렬화/역직렬화하는지 정리해보겠습니다.

