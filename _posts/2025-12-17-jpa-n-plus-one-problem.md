---
layout: post
title: "JPA의 N+1 문제와 해결 방법"
date: 2025-12-17
categories: [programming, jpa]
tags: [JPA, N+1문제, 성능최적화, FetchJoin, LazyLoading, EagerLoading, BatchSize]
---

# JPA의 N+1 문제와 해결 방법

JPA를 사용하다 보면 가장 흔히 마주치는 성능 문제 중 하나가 **N+1 문제**입니다.  
이 문제는 데이터베이스 쿼리가 예상보다 훨씬 많이 실행되어 성능 저하를 일으킵니다.

이 글에서는 N+1 문제가 무엇인지, 왜 발생하는지, 그리고 어떻게 해결할 수 있는지 정리해보겠습니다.

---

## 1. N+1 문제란?

### 문제 정의

**N+1 문제**는 다음과 같은 상황에서 발생합니다:

1. **1번의 쿼리**로 N개의 엔티티를 조회
2. 각 엔티티의 연관 관계를 조회하기 위해 **추가로 N번의 쿼리** 실행
3. 총 **1 + N번의 쿼리**가 실행됨

### 간단한 예시

```java
@Entity
public class Order {
    @Id
    @GeneratedValue
    private Long orderId;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id")
    private User user;  // 연관 관계
    
    private LocalDateTime orderDate;
}

@Entity
public class User {
    @Id
    @GeneratedValue
    private Long userId;
    
    private String name;
    
    @OneToMany(mappedBy = "user")
    private List<Order> orders;
}
```

**N+1 문제 발생 코드:**
```java
// 1번의 쿼리: 모든 주문 조회
List<Order> orders = orderRepository.findAll();  // SELECT * FROM orders

// N번의 쿼리: 각 주문의 사용자 조회
for (Order order : orders) {
    User user = order.getUser();  // SELECT * FROM users WHERE user_id = ?
    // 각 주문마다 쿼리 실행!
}
```

**실행되는 쿼리:**
```sql
-- 1번째 쿼리
SELECT * FROM orders;

-- 2번째 쿼리 (주문이 10개면 10번 실행)
SELECT * FROM users WHERE user_id = 1;
SELECT * FROM users WHERE users WHERE user_id = 2;
SELECT * FROM users WHERE user_id = 3;
-- ... 총 11번의 쿼리 실행 (1 + 10)
```

---

## 2. N+1 문제가 발생하는 원인

### 원인 1: Lazy Loading (지연 로딩)

**Lazy Loading의 동작:**
- 연관 관계가 `FetchType.LAZY`로 설정되어 있으면, 처음에는 프록시 객체만 생성
- 실제로 연관 엔티티에 접근할 때 데이터베이스에서 조회

```java
@Entity
public class Order {
    @ManyToOne(fetch = FetchType.LAZY)  // 지연 로딩
    private User user;
}

// 사용
List<Order> orders = orderRepository.findAll();  // 1번의 쿼리
for (Order order : orders) {
    order.getUser().getName();  // 각 주문마다 쿼리 실행 (N번)
}
```

### 원인 2: Eager Loading (즉시 로딩)

**Eager Loading의 문제:**
- `FetchType.EAGER`로 설정해도 N+1 문제가 발생할 수 있음
- JPA는 연관 관계를 조인하지 않고 별도 쿼리로 조회할 수 있음

```java
@Entity
public class Order {
    @ManyToOne(fetch = FetchType.EAGER)  // 즉시 로딩
    private User user;
}

// findAll() 호출 시
List<Order> orders = orderRepository.findAll();
// 1번의 쿼리로 주문 조회
// 그 다음 각 주문의 사용자를 조회하기 위해 N번의 쿼리 실행
```

### 원인 3: 컬렉션 연관 관계

**@OneToMany, @ManyToMany에서도 동일하게 발생:**

```java
@Entity
public class User {
    @OneToMany(mappedBy = "user", fetch = FetchType.LAZY)
    private List<Order> orders;
}

// 사용
List<User> users = userRepository.findAll();  // 1번의 쿼리
for (User user : users) {
    List<Order> orders = user.getOrders();  // 각 사용자마다 쿼리 실행 (N번)
}
```

---

## 3. N+1 문제 확인 방법

### 로그로 확인

**application.yml 설정:**
```yaml
spring:
  jpa:
    properties:
      hibernate:
        show_sql: true
        format_sql: true
    logging:
      level:
        org.hibernate.SQL: DEBUG
        org.hibernate.type.descriptor.sql.BasicBinder: TRACE
```

**실행 로그 확인:**
```
Hibernate: select order0_.order_id as order_id1_0_, ... from orders order0_
Hibernate: select user0_.user_id as user_id1_1_0_ from users user0_ where user0_.user_id=?
Hibernate: select user0_.user_id as user_id1_1_0_ from users user0_ where user0_.user_id=?
Hibernate: select user0_.user_id as user_id1_1_0_ from users user0_ where user0_.user_id=?
-- ... 반복되는 쿼리 패턴 확인
```

### 쿼리 카운터로 확인

```java
@Component
public class QueryCounter implements Interceptor {
    private ThreadLocal<Integer> queryCount = new ThreadLocal<>();
    
    public void start() {
        queryCount.set(0);
    }
    
    public int getCount() {
        return queryCount.get();
    }
    
    // 쿼리 실행 시 카운트 증가
}
```

---

## 4. 해결 방법

### 방법 1: Fetch Join 사용 (가장 권장)

**Fetch Join이란?**
- SQL의 JOIN을 사용하여 연관 엔티티를 한 번에 조회
- 한 번의 쿼리로 모든 데이터를 가져옴

**JPQL 사용:**
```java
@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {
    @Query("SELECT o FROM Order o JOIN FETCH o.user")
    List<Order> findAllWithUser();
}

// 사용
List<Order> orders = orderRepository.findAllWithUser();  // 1번의 쿼리만 실행
for (Order order : orders) {
    order.getUser().getName();  // 추가 쿼리 없음
}
```

**생성되는 SQL:**
```sql
SELECT o.*, u.* 
FROM orders o 
INNER JOIN users u ON o.user_id = u.user_id;
```

**컬렉션 Fetch Join:**
```java
@Query("SELECT u FROM User u JOIN FETCH u.orders")
List<User> findAllWithOrders();
```

**주의사항:**
- Fetch Join은 페이징(`Pageable`)과 함께 사용할 수 없음
- 여러 컬렉션을 동시에 Fetch Join하면 카테시안 곱 문제 발생

### 방법 2: @EntityGraph 사용

**@EntityGraph란?**
- JPA 2.1부터 지원
- Fetch Join과 유사하지만 메서드 이름 기반 쿼리에서도 사용 가능

**사용 방법:**
```java
@Entity
@NamedEntityGraph(
    name = "Order.withUser",
    attributeNodes = @NamedAttributeNode("user")
)
public class Order {
    // ...
}

// Repository
@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {
    @EntityGraph("Order.withUser")
    List<Order> findAll();
    
    // 또는 인라인으로 정의
    @EntityGraph(attributePaths = {"user"})
    Optional<Order> findById(Long id);
}
```

**장점:**
- 메서드 이름 기반 쿼리에서도 사용 가능
- 여러 EntityGraph를 정의하여 재사용 가능

### 방법 3: Batch Size 설정

**@BatchSize 사용:**
- 연관 관계를 조회할 때 한 번에 여러 개를 조회
- N번의 쿼리를 N/batchSize번으로 줄임

**엔티티 레벨 설정:**
```java
@Entity
@BatchSize(size = 10)
public class User {
    @OneToMany(mappedBy = "user")
    private List<Order> orders;
}
```

**연관 관계 레벨 설정:**
```java
@Entity
public class Order {
    @ManyToOne(fetch = FetchType.LAZY)
    @BatchSize(size = 10)
    private User user;
}
```

**동작 방식:**
```java
List<Order> orders = orderRepository.findAll();  // 1번의 쿼리

// 첫 번째 주문의 사용자 접근
orders.get(0).getUser();  
// SELECT * FROM users WHERE user_id IN (1, 2, 3, ..., 10)  -- 10개씩 조회

// 11번째 주문의 사용자 접근
orders.get(10).getUser();
// SELECT * FROM users WHERE user_id IN (11, 12, 13, ..., 20)  -- 다음 10개 조회
```

**전역 설정 (application.yml):**
```yaml
spring:
  jpa:
    properties:
      hibernate:
        jdbc:
          batch_size: 10
```

### 방법 4: DTO Projection 사용

**DTO로 필요한 데이터만 조회:**
```java
public interface OrderSummary {
    Long getOrderId();
    LocalDateTime getOrderDate();
    String getUserName();  // User의 name
}

@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {
    @Query("SELECT o.orderId as orderId, o.orderDate as orderDate, u.name as userName " +
           "FROM Order o JOIN o.user u")
    List<OrderSummary> findAllOrderSummaries();
}
```

**장점:**
- 필요한 데이터만 조회하여 성능 최적화
- N+1 문제 완전 해결

**단점:**
- DTO 클래스 추가 필요
- 엔티티가 아닌 DTO를 반환하므로 영속성 컨텍스트 관리 불가

### 방법 5: 별도 쿼리로 조회 후 매핑

**두 번의 쿼리로 해결:**
```java
@Service
public class OrderService {
    public List<OrderDTO> getOrdersWithUsers() {
        // 1. 주문 조회
        List<Order> orders = orderRepository.findAll();
        
        // 2. 사용자 ID 수집
        Set<Long> userIds = orders.stream()
            .map(order -> order.getUser().getId())
            .collect(Collectors.toSet());
        
        // 3. 사용자 일괄 조회 (1번의 쿼리)
        Map<Long, User> users = userRepository.findAllById(userIds)
            .stream()
            .collect(Collectors.toMap(User::getId, Function.identity()));
        
        // 4. 메모리에서 매핑
        return orders.stream()
            .map(order -> OrderDTO.of(order, users.get(order.getUser().getId())))
            .collect(Collectors.toList());
    }
}
```

---

## 5. 실전 예시

### 예시 1: 단일 연관 관계 (ManyToOne)

**문제 코드:**
```java
List<Order> orders = orderRepository.findAll();
for (Order order : orders) {
    System.out.println(order.getUser().getName());  // N+1 문제
}
```

**해결 방법 1: Fetch Join**
```java
@Query("SELECT o FROM Order o JOIN FETCH o.user")
List<Order> findAllWithUser();
```

**해결 방법 2: @EntityGraph**
```java
@EntityGraph(attributePaths = {"user"})
List<Order> findAll();
```

### 예시 2: 컬렉션 연관 관계 (OneToMany)

**문제 코드:**
```java
List<User> users = userRepository.findAll();
for (User user : users) {
    List<Order> orders = user.getOrders();  // N+1 문제
}
```

**해결 방법 1: Fetch Join**
```java
@Query("SELECT DISTINCT u FROM User u JOIN FETCH u.orders")
List<User> findAllWithOrders();
```

**주의: DISTINCT 사용**
- Fetch Join 시 카테시안 곱으로 인한 중복 발생
- `DISTINCT`로 중복 제거

**해결 방법 2: @BatchSize**
```java
@Entity
@BatchSize(size = 10)
public class User {
    @OneToMany(mappedBy = "user")
    private List<Order> orders;
}
```

### 예시 3: 중첩 연관 관계

**문제 코드:**
```java
List<Order> orders = orderRepository.findAll();
for (Order order : orders) {
    User user = order.getUser();
    List<Address> addresses = user.getAddresses();  // N+1 문제
}
```

**해결 방법: 여러 단계 Fetch Join**
```java
@Query("SELECT o FROM Order o " +
       "JOIN FETCH o.user u " +
       "JOIN FETCH u.addresses")
List<Order> findAllWithUserAndAddresses();
```

### 예시 4: 페이징과 함께 사용

**문제:**
- Fetch Join은 페이징과 함께 사용할 수 없음

**해결 방법: @BatchSize 사용**
```java
@Entity
@BatchSize(size = 20)
public class User {
    @OneToMany(mappedBy = "user")
    private List<Order> orders;
}

// 페이징과 함께 사용
Page<User> users = userRepository.findAll(PageRequest.of(0, 10));
// 각 사용자의 주문은 @BatchSize로 최적화되어 조회됨
```

---

## 6. Fetch Join vs @EntityGraph vs @BatchSize

### 비교표

| 방법 | 쿼리 수 | 페이징 지원 | 중복 데이터 | 사용 시기 |
|------|---------|------------|------------|----------|
| **Fetch Join** | 1번 | ❌ | 가능 (DISTINCT 필요) | 연관 관계를 항상 함께 조회할 때 |
| **@EntityGraph** | 1번 | ✅ (제한적) | 가능 | 메서드 이름 기반 쿼리에서 사용 |
| **@BatchSize** | 1 + N/batchSize | ✅ | 없음 | 페이징이 필요하거나 선택적 로딩 시 |

### 선택 가이드

**Fetch Join을 사용하는 경우:**
- 연관 관계를 항상 함께 조회해야 할 때
- 페이징이 필요 없을 때
- 성능이 가장 중요할 때

**@EntityGraph를 사용하는 경우:**
- 메서드 이름 기반 쿼리에서 사용하고 싶을 때
- 여러 EntityGraph를 재사용하고 싶을 때

**@BatchSize를 사용하는 경우:**
- 페이징이 필요할 때
- 연관 관계를 선택적으로 로딩할 때
- Fetch Join으로 해결할 수 없을 때

---

## 7. 주의사항 및 Best Practice

### DO (해야 할 것)

1. **Fetch Join 우선 사용**
   ```java
   @Query("SELECT o FROM Order o JOIN FETCH o.user")
   List<Order> findAllWithUser();
   ```

2. **@BatchSize로 보완**
   ```java
   @BatchSize(size = 10)
   ```

3. **쿼리 로그 확인**
   ```yaml
   spring.jpa.show-sql: true
   ```

4. **성능 테스트**
   - 실제 데이터로 성능 측정
   - 쿼리 실행 횟수 확인

### DON'T (하지 말아야 할 것)

1. **불필요한 Eager Loading 금지**
   ```java
   // ❌
   @ManyToOne(fetch = FetchType.EAGER)
   ```

2. **컬렉션 Fetch Join 시 페이징 금지**
   ```java
   // ❌
   @Query("SELECT o FROM Order o JOIN FETCH o.items")
   Page<Order> findAll(Pageable pageable);  // 예외 발생
   ```

3. **여러 컬렉션 동시 Fetch Join 주의**
   ```java
   // ⚠️ 카테시안 곱 문제 발생 가능
   @Query("SELECT u FROM User u JOIN FETCH u.orders JOIN FETCH u.addresses")
   ```

4. **트랜잭션 밖에서 Lazy Loading 금지**
   ```java
   // ❌
   @Transactional(readOnly = true)
   public List<Order> getOrders() {
       return orderRepository.findAll();  // 트랜잭션 종료 후 user 접근 시 LazyInitializationException
   }
   ```

---

## 8. 성능 최적화 팁

### 쿼리 최적화

**인덱스 확인:**
```sql
-- 외래 키에 인덱스가 있는지 확인
SHOW INDEX FROM orders;

-- 인덱스가 없으면 생성
CREATE INDEX idx_orders_user_id ON orders(user_id);
```

**SELECT 절 최적화:**
```java
// 필요한 컬럼만 조회
@Query("SELECT o.orderId, o.orderDate, u.name " +
       "FROM Order o JOIN o.user u")
List<OrderSummary> findOrderSummaries();
```

### 캐싱 활용

**2차 캐시 사용:**
```java
@Entity
@Cacheable
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class User {
    // ...
}
```

**설정:**
```yaml
spring:
  jpa:
    properties:
      hibernate:
        cache:
          use_second_level_cache: true
          region:
            factory_class: org.hibernate.cache.jcache.JCacheRegionFactory
```

---

## 마무리

**핵심 포인트:**

- **N+1 문제**: 1번의 쿼리로 N개 조회 후, 각각의 연관 관계 조회로 추가 N번의 쿼리 실행
- **발생 원인**: Lazy Loading, Eager Loading, 컬렉션 연관 관계
- **해결 방법**: Fetch Join (가장 권장), @EntityGraph, @BatchSize, DTO Projection
- **선택 가이드**: 페이징 필요 시 @BatchSize, 항상 함께 조회 시 Fetch Join

N+1 문제는 JPA를 사용할 때 가장 흔히 발생하는 성능 문제입니다. **Fetch Join을 우선적으로 사용**하고, 페이징이 필요한 경우 **@BatchSize**를 활용하여 해결할 수 있습니다. 쿼리 로그를 확인하여 문제를 조기에 발견하고 해결하는 것이 중요합니다. 🚀

다음 글에서는 JPA의 **트랜잭션 관리**와 `@Transactional`의 동작 원리에 대해 정리해보겠습니다.

