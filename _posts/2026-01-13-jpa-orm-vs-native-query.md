---
layout: post
title: "JPA ORM vs 네이티브 쿼리: 효율성과 선택 기준"
date: 2026-01-13
categories: [programming, jpa, database]
tags: [JPA, ORM, NativeQuery, 네이티브쿼리, 성능최적화, FetchJoin, DTOProjection, N+1문제]
---

# JPA ORM vs 네이티브 쿼리: 효율성과 선택 기준

이전 글에서 JPA의 N+1 문제와 해결 방법(Fetch Join, @EntityGraph, @BatchSize 등)을 다뤘습니다. 이번 글에서는 **JPA ORM과 네이티브 쿼리의 효율성 차이**를 비교하고, 각각의 장단점과 실전에서 어떤 상황에 어떤 방식을 선택해야 하는지 정리해보겠습니다.

JPA를 사용하다 보면 "언제 ORM을 사용하고, 언제 네이티브 쿼리를 사용해야 할까?"라는 질문을 자주 마주하게 됩니다. 특히 N+1 문제를 해결하는 여러 방법들과 네이티브 쿼리 중 어떤 것이 더 효율적인지 명확히 이해하는 것이 중요합니다.

---

## 1. JPA ORM과 네이티브 쿼리란?

### 1.1 JPA ORM (Object-Relational Mapping)

**JPA ORM:**
- Java 객체와 관계형 데이터베이스 테이블을 자동으로 매핑
- 객체 지향적 코드로 DB 작업 수행
- Hibernate가 SQL을 자동 생성

**예시:**

```java
@Entity
public class Order {
    @Id
    private Long id;
    
    @ManyToOne(fetch = FetchType.LAZY)
    private User user;
    
    @OneToMany(mappedBy = "order")
    private List<OrderItem> orderItems;
}

// Repository
public interface OrderRepository extends JpaRepository<Order, Long> {
    List<Order> findByStatus(String status);  // 메서드명으로 쿼리 자동 생성
}
```

**특징:**
- 타입 안전성 (컴파일 타임 체크)
- DB 벤더 독립성
- 객체 지향적 코드

### 1.2 네이티브 쿼리 (Native Query)

**네이티브 쿼리:**
- 직접 SQL을 작성하여 실행
- DB 벤더별 SQL 문법 사용 가능
- 실행 계획 직접 제어

**예시:**

```java
@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {
    @Query(value = "SELECT * FROM orders WHERE status = ?1", nativeQuery = true)
    List<Order> findByStatusNative(String status);
    
    @Query(value = """
        SELECT o.*, u.name, u.email
        FROM orders o
        INNER JOIN users u ON o.user_id = u.id
        WHERE o.status = ?1
        ORDER BY o.created_at DESC
        LIMIT ?2
        """, nativeQuery = true)
    List<Object[]> findOrdersWithUserNative(String status, int limit);
}
```

**특징:**
- SQL 직접 제어
- DB 특화 기능 활용 가능
- 성능 최적화 용이

---

## 2. N+1 문제 해결 방법 vs 네이티브 쿼리 효율성 비교

### 2.1 쿼리 실행 횟수 비교

**N+1 문제 상황:**

```java
// ❌ N+1 문제 발생
List<Order> orders = orderRepository.findAll();  // 1번 쿼리
for (Order order : orders) {
    order.getUser().getName();  // N번 쿼리 (각 Order마다)
}
// 총 1 + N번 쿼리 실행
```

**JPA 해결 방법별 쿼리 횟수:**

**1. Fetch Join:**
```java
@Query("SELECT o FROM Order o JOIN FETCH o.user")
List<Order> findAllWithUser();
```
- **쿼리 횟수: 1번** ✅
- JOIN으로 한 번에 조회

**2. @BatchSize:**
```java
@BatchSize(size = 10)
@OneToMany(mappedBy = "order")
private List<OrderItem> orderItems;
```
- **쿼리 횟수: N/batchSize번** (예: 100개 → 10번)
- 배치로 묶어서 조회

**3. DTO Projection:**
```java
@Query("SELECT o.id as orderId, o.date as orderDate, u.name as userName " +
       "FROM Order o JOIN o.user u")
List<OrderSummary> findAllSummaries();
```
- **쿼리 횟수: 1번** ✅
- 필요한 데이터만 조회

**네이티브 쿼리:**
```java
@Query(value = """
    SELECT o.*, u.name, u.email
    FROM orders o
    INNER JOIN users u ON o.user_id = u.id
    """, nativeQuery = true)
List<Object[]> findAllWithUserNative();
```
- **쿼리 횟수: 1번** ✅
- 직접 SQL 작성

**결론:**
- Fetch Join, DTO Projection, 네이티브 쿼리 모두 **1번의 쿼리**로 해결
- @BatchSize는 **여러 번의 쿼리**가 필요하지만, 배치 크기에 따라 최적화 가능

---

### 2.2 메모리 사용량 비교

**JPA Fetch Join의 문제:**

```java
@Query("SELECT o FROM Order o JOIN FETCH o.user JOIN FETCH o.orderItems")
List<Order> findAllWithDetails();
```

**카테시안 곱 발생:**
- Order 100개, 각각 OrderItem 10개
- 결과: 100 × 10 = **1,000개 행** 반환
- 중복 데이터로 메모리 사용량 증가

**예시:**
```
Order(id=1, user=User(id=1), orderItems=[Item1, Item2, Item3])
Order(id=1, user=User(id=1), orderItems=[Item1, Item2, Item3])  // 중복
Order(id=1, user=User(id=1), orderItems=[Item1, Item2, Item3])  // 중복
...
```

**네이티브 쿼리:**

```sql
SELECT o.*, u.*, oi.*
FROM orders o
LEFT JOIN users u ON o.user_id = u.id
LEFT JOIN order_items oi ON o.id = oi.order_id
WHERE o.created_at >= ?
```

**동일한 카테시안 곱 문제 발생하지만:**
- 필요한 컬럼만 선택 가능
- 예: `SELECT o.id, o.date, u.name`만 조회 → 메모리 절약

**DTO Projection:**

```java
public interface OrderSummary {
    Long getOrderId();
    LocalDateTime getOrderDate();
    String getUserName();
}

@Query("SELECT o.id as orderId, o.date as orderDate, u.name as userName " +
       "FROM Order o JOIN o.user u")
List<OrderSummary> findAllSummaries();
```

**장점:**
- 필요한 데이터만 조회 → **메모리 효율적**
- 카테시안 곱 문제 없음 (필요한 컬럼만)

**비교표:**

| 방법 | 메모리 사용량 | 카테시안 곱 문제 |
|------|--------------|----------------|
| Fetch Join | 높음 | 발생 |
| @BatchSize | 낮음 | 없음 |
| DTO Projection | 낮음 | 없음 |
| 네이티브 쿼리 | 중간~낮음 | 발생 (컬럼 선택 가능) |

**결론:**
- **DTO Projection이 가장 메모리 효율적**
- 네이티브 쿼리는 필요한 컬럼만 선택하면 메모리 효율적
- Fetch Join은 카테시안 곱으로 메모리 사용량 증가

---

### 2.3 네트워크 트래픽 비교

**JPA Fetch Join:**

```java
@Query("SELECT o FROM Order o JOIN FETCH o.user")
List<Order> findAllWithUser();
```

**문제:**
- 엔티티의 **모든 필드**를 조회
- 예: User 엔티티에 `password`, `salt`, `refreshToken` 등 불필요한 필드도 조회
- 네트워크 트래픽 증가

**네이티브 쿼리:**

```sql
SELECT o.id, o.date, o.status, u.name, u.email
FROM orders o
INNER JOIN users u ON o.user_id = u.id
```

**장점:**
- **필요한 컬럼만 명시적으로 선택**
- 네트워크 트래픽 최소화
- 예: `password`, `salt` 등 제외 가능

**DTO Projection:**

```java
@Query("SELECT o.id as orderId, o.date as orderDate, u.name as userName " +
       "FROM Order o JOIN o.user u")
List<OrderSummary> findAllSummaries();
```

**장점:**
- 필요한 데이터만 조회 → 네이티브 쿼리와 유사한 효율

**비교:**

| 방법 | 네트워크 트래픽 | 컬럼 선택 |
|------|----------------|----------|
| Fetch Join | 높음 | 불가 (전체 필드) |
| @BatchSize | 중간 | 불가 (전체 필드) |
| DTO Projection | 낮음 | 가능 |
| 네이티브 쿼리 | 낮음 | 가능 (명시적) |

**결론:**
- **네이티브 쿼리 = DTO Projection > Fetch Join**
- 네이티브 쿼리는 필요한 컬럼만 선택하여 네트워크 트래픽 최소화

---

### 2.4 성능 (실행 계획) 비교

**JPA Fetch Join:**

```java
@Query("SELECT o FROM Order o JOIN FETCH o.user WHERE o.status = :status")
List<Order> findByStatus(@Param("status") String status);
```

**생성되는 SQL:**
```sql
SELECT o.*, u.*
FROM orders o
INNER JOIN users u ON o.user_id = u.id
WHERE o.status = ?
```

**제한사항:**
- Hibernate가 SQL 생성 → 최적화 여지 제한
- 인덱스 힌트 직접 지정 어려움
- 실행 계획 제어 불가

**네이티브 쿼리:**

```sql
SELECT /*+ USE_INDEX(orders, idx_status_created_at) */
       o.*, u.*
FROM orders o
INNER JOIN users u ON o.user_id = u.id
WHERE o.status = ?
  AND o.created_at >= ?
ORDER BY o.created_at DESC
LIMIT 100
```

**장점:**
- **실행 계획 직접 제어 가능**
- 인덱스 힌트 사용 가능
  - MySQL: `USE INDEX`, `FORCE INDEX`
  - PostgreSQL: `/*+ INDEX(...) */`
  - Oracle: `/*+ INDEX(...) */`
- 복잡한 조건 최적화 용이

**예시: 인덱스 힌트**

```sql
-- MySQL
SELECT * FROM orders USE INDEX (idx_status_created_at)
WHERE status = 'PENDING' AND created_at >= '2024-01-01';

-- PostgreSQL
SELECT /*+ INDEX(orders idx_status_created_at) */ *
FROM orders
WHERE status = 'PENDING' AND created_at >= '2024-01-01';
```

**결론:**
- **네이티브 쿼리가 성능 튜닝에 유리**
- 인덱스 힌트, 실행 계획 제어 가능
- 복잡한 쿼리 최적화 용이

---

### 2.5 페이징 처리 비교

**JPA Fetch Join + 페이징:**

```java
@Query("SELECT o FROM Order o JOIN FETCH o.user")
Page<Order> findAllWithUser(Pageable pageable);  // ❌ 에러 발생!
```

**문제:**
- Fetch Join과 페이징을 함께 사용할 수 없음
- Hibernate 에러: "firstResult/maxResults specified with collection fetch"

**해결 방법 1: 별도 쿼리 2번 실행**

```java
// 1. 페이징된 Order ID 조회
@Query("SELECT o.id FROM Order o WHERE o.status = :status")
Page<Long> findOrderIdsByStatus(@Param("status") String status, Pageable pageable);

// 2. Fetch Join으로 상세 정보 조회
@Query("SELECT o FROM Order o JOIN FETCH o.user WHERE o.id IN :ids")
List<Order> findByIdsWithUser(@Param("ids") List<Long> ids);
```

**문제:**
- **2번의 쿼리** 실행 필요
- 복잡한 코드

**해결 방법 2: @BatchSize 사용**

```java
@BatchSize(size = 10)
@OneToMany(mappedBy = "order")
private List<OrderItem> orderItems;

Page<Order> orders = orderRepository.findAll(pageable);  // 1번 쿼리
// 각 Order의 user 접근 시 배치로 조회
```

**특징:**
- 페이징 가능하지만, 추가 쿼리 발생 (배치 크기에 따라)

**네이티브 쿼리 + 페이징:**

```sql
SELECT o.*, u.*
FROM orders o
INNER JOIN users u ON o.user_id = u.id
WHERE o.status = ?
ORDER BY o.created_at DESC
LIMIT 20 OFFSET 0
```

**장점:**
- **페이징 직접 제어 가능**
- **1번의 쿼리**로 해결
- 간단하고 명확

**비교:**

| 방법 | 페이징 가능 | 쿼리 횟수 | 복잡도 |
|------|-----------|----------|--------|
| Fetch Join | 불가 | - | - |
| @BatchSize | 가능 | 1 + N/batchSize | 중간 |
| DTO Projection | 가능 | 1 | 낮음 |
| 네이티브 쿼리 | 가능 | 1 | 낮음 |

**결론:**
- **네이티브 쿼리가 페이징에서 가장 효율적**
- 1번의 쿼리로 페이징 처리 가능
- Fetch Join은 페이징과 함께 사용 불가

---

### 2.6 복잡한 집계 쿼리 비교

**JPA JPQL 제한사항:**

```java
// ❌ 윈도우 함수 사용 불가
@Query("SELECT o, ROW_NUMBER() OVER (PARTITION BY o.userId ORDER BY o.createdAt) " +
       "FROM Order o")
```

**제한사항:**
- 윈도우 함수 (ROW_NUMBER, RANK, DENSE_RANK 등) 제한적
- CTE (Common Table Expression) 사용 어려움
- 복잡한 서브쿼리 표현 제한적

**네이티브 쿼리:**

```sql
WITH monthly_stats AS (
  SELECT 
    DATE_TRUNC('month', created_at) as month,
    COUNT(*) as order_count,
    SUM(total_amount) as total_revenue,
    AVG(total_amount) as avg_order_value
  FROM orders
  WHERE created_at >= '2024-01-01'
  GROUP BY DATE_TRUNC('month', created_at)
)
SELECT 
  month,
  order_count,
  total_revenue,
  avg_order_value,
  LAG(total_revenue) OVER (ORDER BY month) as prev_month_revenue,
  (total_revenue - LAG(total_revenue) OVER (ORDER BY month)) / 
    LAG(total_revenue) OVER (ORDER BY month) * 100 as growth_rate
FROM monthly_stats
ORDER BY month DESC;
```

**장점:**
- **CTE, 윈도우 함수 등 DB 특화 기능 활용 가능**
- 복잡한 집계 쿼리 작성 가능
- 성능 최적화 용이

**결론:**
- **복잡한 집계는 네이티브 쿼리가 필수**
- CTE, 윈도우 함수 등 고급 SQL 기능 활용 가능

---

## 3. 종합 비교표

| 구분 | Fetch Join | @BatchSize | DTO Projection | 네이티브 쿼리 |
|------|-----------|------------|----------------|--------------|
| **쿼리 횟수** | 1번 ✅ | N/batchSize번 | 1번 ✅ | 1번 ✅ |
| **메모리 사용** | 높음 (카테시안 곱) | 낮음 | 낮음 ✅ | 낮음 ✅ |
| **네트워크 트래픽** | 높음 (전체 필드) | 중간 | 낮음 ✅ | 낮음 ✅ |
| **성능 최적화** | 제한적 | 제한적 | 제한적 | 우수 ✅ |
| **페이징** | 불가 ❌ | 가능 (추가 쿼리) | 가능 ✅ | 가능 ✅ |
| **복잡한 쿼리** | 제한적 | 불가 | 제한적 | 우수 ✅ |
| **타입 안전성** | 우수 ✅ | 우수 ✅ | 중간 | 낮음 |
| **DB 독립성** | 우수 ✅ | 우수 ✅ | 우수 ✅ | 낮음 |
| **유지보수성** | 우수 ✅ | 우수 ✅ | 중간 | 낮음 |
| **코드 간결성** | 우수 ✅ | 우수 ✅ | 중간 | 낮음 |

---

## 4. 실전 사용 시나리오

### 4.1 JPA ORM을 사용해야 하는 경우

**1. 단순 CRUD 작업:**

```java
// ✅ ORM 사용
public interface OrderRepository extends JpaRepository<Order, Long> {
    List<Order> findByStatus(String status);
    Optional<Order> findById(Long id);
    void save(Order order);
}
```

**2. 객체 관계 중심 비즈니스 로직:**

```java
// ✅ ORM 사용 (변경 감지 활용)
@Transactional
public void updateOrderStatus(Long orderId, String status) {
    Order order = orderRepository.findById(orderId).orElseThrow();
    order.setStatus(status);  // 변경 감지로 자동 업데이트
    // orderRepository.save() 불필요
}
```

**3. 빠른 프로토타이핑:**

```java
// ✅ ORM 사용 (빠른 개발)
List<Order> orders = orderRepository.findByUserIdAndStatus(userId, "PENDING");
```

**4. DB 벤더 독립성이 중요한 경우:**

```java
// ✅ ORM 사용 (MySQL, PostgreSQL, H2 모두 동작)
@Query("SELECT o FROM Order o WHERE o.status = :status")
List<Order> findByStatus(@Param("status") String status);
```

### 4.2 네이티브 쿼리를 사용해야 하는 경우

**1. 복잡한 집계 쿼리:**

```java
// ✅ 네이티브 쿼리 사용
@Query(value = """
    SELECT 
        DATE_TRUNC('month', created_at) as month,
        COUNT(*) as order_count,
        SUM(total_amount) as total_revenue
    FROM orders
    WHERE created_at >= ?1
    GROUP BY DATE_TRUNC('month', created_at)
    ORDER BY month DESC
    """, nativeQuery = true)
List<Object[]> findMonthlyStats(LocalDateTime startDate);
```

**2. 대량 데이터 처리 (성능 중요):**

```java
// ✅ 네이티브 쿼리 사용 (인덱스 힌트)
@Query(value = """
    SELECT /*+ USE_INDEX(orders, idx_status_created_at) */
           o.*, u.name
    FROM orders o
    INNER JOIN users u ON o.user_id = u.id
    WHERE o.status = ?1
      AND o.created_at >= ?2
    ORDER BY o.created_at DESC
    LIMIT ?3
    """, nativeQuery = true)
List<Object[]> findRecentOrders(String status, LocalDateTime startDate, int limit);
```

**3. DB 특화 기능 필요:**

```java
// ✅ 네이티브 쿼리 사용 (PostgreSQL Full-Text Search)
@Query(value = """
    SELECT o.*, 
           ts_rank(to_tsvector('english', o.description), 
                   plainto_tsquery('english', ?1)) as rank
    FROM orders o
    WHERE to_tsvector('english', o.description) @@ plainto_tsquery('english', ?1)
    ORDER BY rank DESC
    LIMIT 100
    """, nativeQuery = true)
List<Object[]> searchOrders(String query);
```

**4. 기존 SQL 재사용:**

```java
// ✅ 네이티브 쿼리 사용 (기존 SQL 쿼리 재사용)
@Query(value = "SELECT * FROM complex_report_view WHERE date >= ?1", 
       nativeQuery = true)
List<Object[]> findReportData(LocalDate date);
```

### 4.3 하이브리드 접근법 (권장)

**80/20 법칙:**
- **80%는 JPA ORM 사용** (단순 쿼리, CRUD)
- **20%는 네이티브 쿼리 사용** (복잡한 집계, 성능 병목)

**예시:**

```java
@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {
    // ✅ ORM 사용 (단순 조회)
    List<Order> findByStatus(String status);
    Optional<Order> findById(Long id);
    
    // ✅ Fetch Join 사용 (N+1 해결)
    @Query("SELECT o FROM Order o JOIN FETCH o.user WHERE o.status = :status")
    List<Order> findByStatusWithUser(@Param("status") String status);
    
    // ✅ DTO Projection 사용 (필요한 데이터만)
    @Query("SELECT o.id as orderId, o.date as orderDate, u.name as userName " +
           "FROM Order o JOIN o.user u WHERE o.status = :status")
    List<OrderSummary> findSummariesByStatus(@Param("status") String status);
    
    // ✅ 네이티브 쿼리 사용 (복잡한 집계)
    @Query(value = """
        SELECT 
            DATE_TRUNC('month', created_at) as month,
            COUNT(*) as order_count,
            SUM(total_amount) as total_revenue
        FROM orders
        WHERE created_at >= ?1
        GROUP BY DATE_TRUNC('month', created_at)
        """, nativeQuery = true)
    List<Object[]> findMonthlyStats(LocalDateTime startDate);
}
```

---

## 5. 성능 벤치마크 예시

### 5.1 단순 조회 (Order 10,000건)

**테스트 시나리오:**
- Order 10,000건 조회
- User 정보 포함

**결과:**

| 방법 | 쿼리 횟수 | 실행 시간 | 메모리 사용 |
|------|----------|----------|------------|
| N+1 (기본) | 10,001번 | 5.2초 | 120MB |
| Fetch Join | 1번 | 0.8초 | 180MB |
| @BatchSize(100) | 101번 | 1.2초 | 130MB |
| DTO Projection | 1번 | 0.6초 | 80MB |
| 네이티브 쿼리 | 1번 | 0.5초 | 75MB |

**결론:**
- **네이티브 쿼리 = DTO Projection > Fetch Join > @BatchSize**
- 메모리 사용량은 DTO Projection과 네이티브 쿼리가 가장 효율적

### 5.2 복잡한 집계 (월별 통계)

**테스트 시나리오:**
- 12개월 월별 주문 통계
- 윈도우 함수 사용

**결과:**

| 방법 | 실행 시간 | 복잡도 |
|------|----------|--------|
| JPQL (제한적) | 구현 불가 | - |
| 네이티브 쿼리 | 0.3초 | 낮음 |

**결론:**
- 복잡한 집계는 **네이티브 쿼리가 필수**

---

## 6. Best Practices

### 6.1 선택 기준 체크리스트

**JPA ORM을 선택해야 하는 경우:**
- [ ] 단순 CRUD 작업
- [ ] 객체 관계 중심 비즈니스 로직
- [ ] DB 벤더 독립성이 중요
- [ ] 빠른 프로토타이핑
- [ ] 타입 안전성이 중요

**네이티브 쿼리를 선택해야 하는 경우:**
- [ ] 복잡한 집계 쿼리 (CTE, 윈도우 함수)
- [ ] 대량 데이터 처리 (성능 중요)
- [ ] DB 특화 기능 필요
- [ ] 인덱스 힌트 등 실행 계획 제어 필요
- [ ] 기존 SQL 쿼리 재사용

### 6.2 네이티브 쿼리 사용 시 주의사항

**1. 타입 안전성 확보:**

```java
// ❌ Object[] 반환 (타입 안전성 부족)
@Query(value = "SELECT * FROM orders", nativeQuery = true)
List<Object[]> findAllNative();

// ✅ DTO로 매핑
@Query(value = """
    SELECT o.id as orderId, o.date as orderDate, u.name as userName
    FROM orders o
    INNER JOIN users u ON o.user_id = u.id
    """, nativeQuery = true)
List<OrderSummaryDTO> findAllWithDTO();
```

**2. SQL Injection 방지:**

```java
// ❌ 위험 (SQL Injection)
@Query(value = "SELECT * FROM orders WHERE status = '" + status + "'", 
       nativeQuery = true)

// ✅ 안전 (파라미터 바인딩)
@Query(value = "SELECT * FROM orders WHERE status = ?1", nativeQuery = true)
List<Order> findByStatus(String status);
```

**3. DB 벤더 종속성 관리:**

```java
// ❌ PostgreSQL 전용
@Query(value = "SELECT * FROM orders WHERE created_at >= NOW() - INTERVAL '1 day'", 
       nativeQuery = true)

// ✅ 프로파일별 분리 또는 함수로 추상화
@Profile("postgresql")
@Query(value = "SELECT * FROM orders WHERE created_at >= NOW() - INTERVAL '1 day'", 
       nativeQuery = true)
List<Order> findRecentOrdersPostgres();

@Profile("mysql")
@Query(value = "SELECT * FROM orders WHERE created_at >= DATE_SUB(NOW(), INTERVAL 1 DAY)", 
       nativeQuery = true)
List<Order> findRecentOrdersMySQL();
```

### 6.3 성능 모니터링

**쿼리 로그 확인:**

```yaml
spring:
  jpa:
    properties:
      hibernate:
        format_sql: true
        show_sql: true
        use_sql_comments: true
```

**실행 계획 확인:**

```sql
-- PostgreSQL
EXPLAIN ANALYZE
SELECT o.*, u.*
FROM orders o
INNER JOIN users u ON o.user_id = u.id
WHERE o.status = 'PENDING';

-- MySQL
EXPLAIN
SELECT o.*, u.*
FROM orders o
INNER JOIN users u ON o.user_id = u.id
WHERE o.status = 'PENDING';
```

---

## 7. 실전 예시: 동일한 요구사항을 다른 방법으로 구현

### 7.1 요구사항

**"최근 100건의 주문을 사용자 정보와 함께 조회"**

### 7.2 방법 1: Fetch Join

```java
@Query("SELECT o FROM Order o JOIN FETCH o.user ORDER BY o.createdAt DESC")
List<Order> findRecentOrdersWithUser(Pageable pageable);
```

**문제:**
- 페이징과 함께 사용 불가
- 모든 필드 조회 (메모리 사용량 증가)

### 7.3 방법 2: @BatchSize

```java
@BatchSize(size = 50)
@ManyToOne(fetch = FetchType.LAZY)
private User user;

Page<Order> orders = orderRepository.findAll(
    PageRequest.of(0, 100, Sort.by("createdAt").descending())
);
// 각 Order의 user 접근 시 배치로 조회
```

**특징:**
- 페이징 가능
- 추가 쿼리 발생 (배치 크기에 따라)

### 7.4 방법 3: DTO Projection

```java
public interface OrderWithUserDTO {
    Long getOrderId();
    LocalDateTime getOrderDate();
    String getStatus();
    String getUserName();
    String getUserEmail();
}

@Query("""
    SELECT o.id as orderId, 
           o.date as orderDate, 
           o.status as status,
           u.name as userName, 
           u.email as userEmail
    FROM Order o 
    JOIN o.user u 
    ORDER BY o.createdAt DESC
    """)
Page<OrderWithUserDTO> findRecentOrdersWithUser(Pageable pageable);
```

**장점:**
- 페이징 가능
- 필요한 데이터만 조회
- 메모리 효율적

### 7.5 방법 4: 네이티브 쿼리

```java
@Query(value = """
    SELECT o.id, o.date, o.status, o.total_amount,
           u.name, u.email
    FROM orders o
    INNER JOIN users u ON o.user_id = u.id
    ORDER BY o.created_at DESC
    LIMIT ?1 OFFSET ?2
    """, nativeQuery = true)
List<Object[]> findRecentOrdersWithUserNative(int limit, int offset);
```

**장점:**
- 페이징 직접 제어
- 필요한 컬럼만 선택
- 성능 최적화 용이 (인덱스 힌트 등)

**비교:**

| 방법 | 페이징 | 쿼리 횟수 | 메모리 | 성능 |
|------|--------|----------|--------|------|
| Fetch Join | 불가 | 1 | 높음 | 중간 |
| @BatchSize | 가능 | 1 + N/batchSize | 중간 | 중간 |
| DTO Projection | 가능 | 1 | 낮음 | 우수 |
| 네이티브 쿼리 | 가능 | 1 | 낮음 | 우수 |

**권장:**
- **DTO Projection** 또는 **네이티브 쿼리** 사용
- 페이징이 필요하고 메모리 효율이 중요할 때

---

## 마무리

**핵심 포인트:**

- **JPA ORM과 네이티브 쿼리는 각각의 장단점이 있으며, 상황에 따라 선택해야 합니다.**
- **N+1 문제 해결 방법 중 Fetch Join, DTO Projection, 네이티브 쿼리는 모두 1번의 쿼리로 해결하지만, 메모리와 네트워크 효율성에서 차이가 있습니다.**
- **단순 조회는 JPA ORM, 복잡한 집계나 성능이 중요한 경우는 네이티브 쿼리를 사용하는 것이 좋습니다.**
- **하이브리드 접근법(80% ORM, 20% 네이티브)을 통해 타입 안전성과 성능을 모두 확보할 수 있습니다.**

**최종 권장사항:**

1. **기본은 JPA ORM 사용** (타입 안전성, 유지보수성)
2. **N+1 문제는 Fetch Join 또는 DTO Projection으로 해결**
3. **복잡한 집계, 대량 데이터, 성능 병목 구간은 네이티브 쿼리 사용**
4. **네이티브 쿼리 사용 시 타입 안전성 확보 (DTO 매핑) 및 SQL Injection 방지**

JPA를 효과적으로 활용하려면 ORM과 네이티브 쿼리의 장단점을 이해하고, 상황에 맞는 최적의 방법을 선택하는 것이 중요합니다. 🚀



