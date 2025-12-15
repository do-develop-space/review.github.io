---
layout: post
title: "JPA에서 Optional을 잘 이용해야 하는 이유"
date: 2025-12-15
categories: [programming, jpa]
tags: [JPA, Optional, Java, Null처리, Repository, BestPractice]
---

# JPA에서 Optional을 잘 이용해야 하는 이유

JPA Repository 메서드에서 `Optional<T>`를 반환하는 것은 매우 일반적입니다.  
하지만 Optional을 단순히 "null을 감싸는 것"으로만 생각하면, 오히려 코드를 더 복잡하고 오류가 발생하기 쉬운 코드로 만들 수 있습니다.

이 글에서는 JPA에서 Optional을 올바르게 사용하는 방법과, 잘못 사용했을 때 발생하는 문제들을 정리해보겠습니다.

---

## 1. JPA에서 Optional이 필요한 이유

### JPA Repository의 Optional 반환

JPA Repository는 기본적으로 `Optional<T>`를 반환하는 메서드를 제공합니다:

```java
public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findById(Long id);
    Optional<User> findByEmail(String email);
    Optional<User> findByUsername(String username);
}
```

**왜 Optional을 사용하는가?**

1. **명시적인 null 가능성**: 메서드 시그니처만 봐도 null이 반환될 수 있음을 알 수 있음
2. **NullPointerException 방지**: Optional을 통해 null 체크를 강제
3. **함수형 프로그래밍 스타일**: map, filter, orElse 등의 메서드로 안전한 처리

### 전통적인 null 처리의 문제점

**Before (null 반환):**
```java
public interface UserRepository {
    User findById(Long id);  // null이 반환될 수 있음
}

// 사용하는 쪽
User user = userRepository.findById(1L);
if (user != null) {
    String email = user.getEmail();  // NPE 위험
}
```

**After (Optional 반환):**
```java
public interface UserRepository {
    Optional<User> findById(Long id);  // 명시적으로 null 가능성 표현
}

// 사용하는 쪽
Optional<User> userOpt = userRepository.findById(1L);
if (userOpt.isPresent()) {
    User user = userOpt.get();
    String email = user.getEmail();
}
```

---

## 2. 잘못된 Optional 사용 예시

### 안티패턴 1: Optional.get()을 바로 호출

```java
// ❌ 잘못된 사용
Optional<User> userOpt = userRepository.findById(1L);
User user = userOpt.get();  // NoSuchElementException 발생 가능!
String email = user.getEmail();
```

**문제점:**
- Optional이 비어있으면 `NoSuchElementException` 발생
- null 체크와 동일한 문제를 그대로 가짐

### 안티패턴 2: isPresent() + get() 패턴

```java
// ❌ 비효율적인 사용
Optional<User> userOpt = userRepository.findById(1L);
if (userOpt.isPresent()) {
    User user = userOpt.get();
    String email = user.getEmail();
    // 처리 로직
}
```

**문제점:**
- Optional의 장점을 전혀 활용하지 못함
- 함수형 스타일을 사용하지 않음

### 안티패턴 3: Optional을 필드나 파라미터로 사용

```java
// ❌ 잘못된 사용
public class User {
    private Optional<String> email;  // Optional을 필드로 사용
    private Optional<Address> address;
}

// ❌ 잘못된 사용
public void processUser(Optional<User> user) {  // Optional을 파라미터로 사용
    // ...
}
```

**문제점:**
- Optional은 반환 타입으로만 사용해야 함
- 필드나 파라미터로 사용하면 불필요한 래핑/언래핑 발생
- 직렬화 문제 발생 가능

### 안티패턴 4: Optional.of()로 null을 감싸기

```java
// ❌ 잘못된 사용
User user = userRepository.findById(1L).orElse(null);
Optional<User> userOpt = Optional.of(user);  // user가 null이면 NPE 발생!

// 올바른 사용
Optional<User> userOpt = Optional.ofNullable(user);  // null 안전
```

---

## 3. 올바른 Optional 사용 방법

### 방법 1: orElse() / orElseGet() 사용

**기본값 제공:**
```java
// orElse(): 항상 기본값 객체 생성 (비용 고려 필요)
User user = userRepository.findById(1L)
    .orElse(new User());  // Optional이 비어있어도 User 객체 생성됨

// orElseGet(): Optional이 비어있을 때만 기본값 생성 (권장)
User user = userRepository.findById(1L)
    .orElseGet(() -> new User());  // Optional이 비어있을 때만 실행

// orElseGet()을 사용해야 하는 경우
User user = userRepository.findById(1L)
    .orElseGet(() -> {
        // 복잡한 기본값 생성 로직
        return createDefaultUser();
    });
```

**성능 차이:**
```java
// ❌ 비효율적: 항상 expensiveOperation() 실행
User user = userRepository.findById(1L)
    .orElse(expensiveOperation());

// ✅ 효율적: Optional이 비어있을 때만 실행
User user = userRepository.findById(1L)
    .orElseGet(() -> expensiveOperation());
```

### 방법 2: orElseThrow() 사용

**예외 발생:**
```java
// 명시적인 예외 처리
User user = userRepository.findById(1L)
    .orElseThrow(() -> new UserNotFoundException("User not found: " + 1L));

// 기본 예외 (NoSuchElementException)
User user = userRepository.findById(1L)
    .orElseThrow();
```

**커스텀 예외:**
```java
public class UserNotFoundException extends RuntimeException {
    public UserNotFoundException(String message) {
        super(message);
    }
}

// 사용
User user = userRepository.findById(1L)
    .orElseThrow(() -> new UserNotFoundException("User not found"));
```

### 방법 3: map() / flatMap() 사용

**값 변환:**
```java
// map(): Optional 내부 값 변환
Optional<String> emailOpt = userRepository.findById(1L)
    .map(User::getEmail);  // User -> String 변환

// 중첩 Optional 처리
Optional<Address> addressOpt = userRepository.findById(1L)
    .map(User::getAddress);  // Optional<Address> 반환

// flatMap(): 중첩 Optional 평탄화
Optional<String> cityOpt = userRepository.findById(1L)
    .map(User::getAddress)
    .flatMap(Address::getCity);  // Optional<Optional<String>> -> Optional<String>
```

**실전 예시:**
```java
// 사용자 이메일의 도메인 추출
Optional<String> domainOpt = userRepository.findByEmail("user@example.com")
    .map(User::getEmail)
    .map(email -> email.split("@")[1]);

// 안전한 체이닝
String result = userRepository.findById(1L)
    .map(User::getAddress)
    .map(Address::getCity)
    .orElse("Unknown");
```

### 방법 4: filter() 사용

**조건부 처리:**
```java
// 활성화된 사용자만 처리
Optional<User> activeUserOpt = userRepository.findById(1L)
    .filter(User::isActive);

// 복잡한 조건
Optional<User> validUserOpt = userRepository.findByEmail("user@example.com")
    .filter(user -> user.isActive() && user.isEmailVerified());
```

**실전 예시:**
```java
// 활성화된 사용자의 이메일만 반환
String email = userRepository.findById(1L)
    .filter(User::isActive)
    .map(User::getEmail)
    .orElseThrow(() -> new InactiveUserException("User is not active"));
```

### 방법 5: ifPresent() / ifPresentOrElse() 사용

**부수 효과(Side Effect) 처리:**
```java
// ifPresent(): 값이 있을 때만 처리
userRepository.findById(1L)
    .ifPresent(user -> {
        sendWelcomeEmail(user);
        logUserActivity(user);
    });

// ifPresentOrElse(): 값이 있을 때와 없을 때 모두 처리
userRepository.findById(1L)
    .ifPresentOrElse(
        user -> sendWelcomeEmail(user),
        () -> logUserNotFound(1L)
    );
```

---

## 4. JPA Repository에서의 Optional 활용

### 커스텀 쿼리와 Optional

```java
public interface UserRepository extends JpaRepository<User, Long> {
    // 단일 결과: Optional 반환
    @Query("SELECT u FROM User u WHERE u.email = :email")
    Optional<User> findByEmail(@Param("email") String email);
    
    // 여러 결과: List 반환 (Optional 사용 안 함)
    @Query("SELECT u FROM User u WHERE u.status = :status")
    List<User> findByStatus(@Param("status") UserStatus status);
}
```

### Optional과 연관 관계

```java
// 연관된 엔티티 조회
Optional<Order> orderOpt = orderRepository.findById(orderId);
Optional<User> userOpt = orderOpt
    .map(Order::getUser);  // Order -> User 변환

// 중첩 연관 관계
Optional<String> userNameOpt = orderRepository.findById(orderId)
    .map(Order::getUser)
    .map(User::getName);
```

### Optional과 Projection

```java
// Projection 인터페이스
public interface UserSummary {
    String getEmail();
    String getName();
}

// Optional과 함께 사용
Optional<UserSummary> summaryOpt = userRepository.findById(1L)
    .map(user -> {
        return new UserSummary() {
            @Override
            public String getEmail() { return user.getEmail(); }
            @Override
            public String getName() { return user.getName(); }
        };
    });
```

---

## 5. 실전 예시: Service 레이어에서의 Optional 활용

### 예시 1: 사용자 조회 및 처리

```java
@Service
public class UserService {
    private final UserRepository userRepository;
    
    // ❌ 잘못된 방법
    public String getUserEmail(Long userId) {
        Optional<User> userOpt = userRepository.findById(userId);
        if (userOpt.isPresent()) {
            return userOpt.get().getEmail();
        }
        return null;  // null 반환은 좋지 않음
    }
    
    // ✅ 올바른 방법 1: 기본값 제공
    public String getUserEmail(Long userId) {
        return userRepository.findById(userId)
            .map(User::getEmail)
            .orElse("unknown@example.com");
    }
    
    // ✅ 올바른 방법 2: 예외 발생
    public String getUserEmail(Long userId) {
        return userRepository.findById(userId)
            .map(User::getEmail)
            .orElseThrow(() -> new UserNotFoundException("User not found: " + userId));
    }
    
    // ✅ 올바른 방법 3: Optional 반환 (호출자가 결정)
    public Optional<String> getUserEmail(Long userId) {
        return userRepository.findById(userId)
            .map(User::getEmail);
    }
}
```

### 예시 2: 복잡한 비즈니스 로직

```java
@Service
public class OrderService {
    private final OrderRepository orderRepository;
    private final UserRepository userRepository;
    
    // 주문의 사용자 이메일로 환불 처리
    public void processRefund(Long orderId) {
        String email = orderRepository.findById(orderId)
            .map(Order::getUser)
            .map(User::getEmail)
            .filter(e -> e != null && !e.isEmpty())
            .orElseThrow(() -> new InvalidOrderException("Order has no valid email"));
        
        refundService.refund(email, orderId);
    }
    
    // 활성화된 사용자의 최근 주문 조회
    public Optional<Order> getRecentOrderForActiveUser(Long userId) {
        return userRepository.findById(userId)
            .filter(User::isActive)
            .flatMap(user -> orderRepository.findFirstByUserOrderByCreatedAtDesc(user));
    }
}
```

### 예시 3: 조건부 업데이트

```java
@Service
public class UserService {
    private final UserRepository userRepository;
    
    // 사용자 이메일 인증 처리
    public void verifyUserEmail(Long userId) {
        userRepository.findById(userId)
            .filter(user -> !user.isEmailVerified())
            .ifPresent(user -> {
                user.verifyEmail();
                userRepository.save(user);
            });
    }
    
    // 사용자 활성화 (조건부)
    public boolean activateUser(Long userId) {
        return userRepository.findById(userId)
            .filter(user -> !user.isActive())
            .map(user -> {
                user.activate();
                userRepository.save(user);
                return true;
            })
            .orElse(false);
    }
}
```

---

## 6. Optional과 성능 고려사항

### orElse() vs orElseGet()

```java
// ❌ 비효율적: 항상 실행됨
User user = userRepository.findById(1L)
    .orElse(createExpensiveUser());  // Optional이 비어있지 않아도 실행

// ✅ 효율적: 필요할 때만 실행
User user = userRepository.findById(1L)
    .orElseGet(() -> createExpensiveUser());  // Optional이 비어있을 때만 실행
```

### Optional 체이닝 최적화

```java
// ❌ 비효율적: 여러 번의 Optional 생성
Optional<User> userOpt = userRepository.findById(1L);
if (userOpt.isPresent()) {
    Optional<Address> addressOpt = Optional.ofNullable(userOpt.get().getAddress());
    if (addressOpt.isPresent()) {
        // 처리
    }
}

// ✅ 효율적: 체이닝으로 한 번에 처리
userRepository.findById(1L)
    .map(User::getAddress)
    .ifPresent(address -> {
        // 처리
    });
```

---

## 7. Optional 사용 가이드라인

### DO (해야 할 것)

1. **Repository 메서드 반환 타입으로 사용**
   ```java
   Optional<User> findById(Long id);
   ```

2. **map(), flatMap(), filter() 활용**
   ```java
   userRepository.findById(1L)
       .map(User::getEmail)
       .filter(email -> email.contains("@"))
       .orElse("default@example.com");
   ```

3. **orElseGet()을 기본값 생성에 사용**
   ```java
   .orElseGet(() -> createDefaultUser());
   ```

4. **orElseThrow()로 명시적 예외 처리**
   ```java
   .orElseThrow(() -> new UserNotFoundException("User not found"));
   ```

### DON'T (하지 말아야 할 것)

1. **필드나 파라미터로 Optional 사용 금지**
   ```java
   // ❌
   private Optional<String> email;
   public void process(Optional<User> user) { }
   ```

2. **Optional.get()을 바로 호출하지 않기**
   ```java
   // ❌
   User user = userOpt.get();
   ```

3. **isPresent() + get() 패턴 피하기**
   ```java
   // ❌
   if (userOpt.isPresent()) {
       User user = userOpt.get();
   }
   ```

4. **Optional.of()로 null 감싸기 금지**
   ```java
   // ❌
   Optional.of(null);  // NPE 발생
   // ✅
   Optional.ofNullable(null);  // 안전
   ```

---

## 8. Optional과 예외 처리

### 예외 처리 전략

```java
@Service
public class UserService {
    // 전략 1: Optional 반환 (호출자가 처리)
    public Optional<User> findUser(Long id) {
        return userRepository.findById(id);
    }
    
    // 전략 2: 예외 발생 (명시적)
    public User getUser(Long id) {
        return userRepository.findById(id)
            .orElseThrow(() -> new UserNotFoundException("User not found: " + id));
    }
    
    // 전략 3: 기본값 제공
    public User getUserOrDefault(Long id) {
        return userRepository.findById(id)
            .orElseGet(() -> createDefaultUser());
    }
}
```

### 커스텀 예외와 Optional

```java
public class UserNotFoundException extends RuntimeException {
    public UserNotFoundException(String message) {
        super(message);
    }
}

// 사용
User user = userRepository.findById(1L)
    .orElseThrow(() -> new UserNotFoundException("User not found"));
```

---

## 마무리

**핵심 포인트:**

- **Optional의 목적**: null 안전성을 제공하고, 명시적으로 null 가능성을 표현
- **올바른 사용**: Repository 반환 타입으로만 사용, map/flatMap/filter 체이닝 활용
- **잘못된 사용**: 필드/파라미터로 사용, get() 바로 호출, isPresent() + get() 패턴
- **성능 고려**: orElseGet()을 기본값 생성에 사용, 불필요한 Optional 생성 피하기

JPA에서 Optional을 올바르게 사용하면 **null 안전성**과 **가독성**을 동시에 확보할 수 있습니다. Optional을 단순히 null을 감싸는 도구가 아니라, **함수형 프로그래밍 스타일로 안전하게 데이터를 처리하는 도구**로 생각해야 합니다. 🎯

Optional과 마찬가지로, JPA 엔티티를 `Set`이나 `Map`에서 사용할 때도 주의가 필요합니다. 다음 글에서는 **equals/hashCode 구현과 영속성 컨텍스트 이슈**에 대해 정리해보겠습니다.

