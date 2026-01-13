---
layout: post
title: "JPA 네이티브 쿼리 완전 가이드: 사용법과 Best Practices"
date: 2026-01-13
categories: [programming, jpa, database]
tags: [JPA, NativeQuery, 네이티브쿼리, SQL, 성능최적화, DTO매핑, SQLInjection]
---

# JPA 네이티브 쿼리 완전 가이드: 사용법과 Best Practices

이전 글에서 JPA의 N+1 문제와 해결 방법을 다뤘습니다. 이번 글에서는 **JPA 네이티브 쿼리(Native Query)**를 사용하는 방법과 주의사항, Best Practices를 정리해보겠습니다.

JPA를 사용하다 보면 복잡한 집계 쿼리, 대량 데이터 처리, DB 특화 기능이 필요할 때가 있습니다. 이때 JPQL로는 표현하기 어렵거나 성능이 떨어지는 경우, 네이티브 쿼리를 사용하여 직접 SQL을 작성할 수 있습니다.

---

## 1. 네이티브 쿼리란?

### 1.1 네이티브 쿼리의 개념

**네이티브 쿼리 (Native Query):**
- JPA에서 **직접 SQL을 작성**하여 실행하는 방법
- DB 벤더별 SQL 문법 사용 가능
- JPQL의 제한사항을 우회하여 복잡한 쿼리 작성 가능

**JPQL vs 네이티브 쿼리:**

```java
// JPQL (Java Persistence Query Language)
@Query("SELECT o FROM Order o WHERE o.status = :status")
List<Order> findByStatus(@Param("status") String status);

// 네이티브 쿼리 (Native SQL)
@Query(value = "SELECT * FROM orders WHERE status = ?1", nativeQuery = true)
List<Order> findByStatusNative(String status);
```

**차이점:**
- **JPQL**: 엔티티와 속성 기반 (객체 지향)
- **네이티브 쿼리**: 테이블과 컬럼 기반 (SQL 직접 작성)

### 1.2 네이티브 쿼리를 사용해야 하는 경우

**1. 복잡한 집계 쿼리:**
- CTE (Common Table Expression)
- 윈도우 함수 (ROW_NUMBER, RANK, DENSE_RANK 등)
- 복잡한 서브쿼리

**2. 대량 데이터 처리:**
- 성능 최적화가 중요한 경우
- 인덱스 힌트 사용 필요
- 실행 계획 직접 제어

**3. DB 특화 기능:**
- PostgreSQL Full-Text Search
- MySQL JSON 함수
- Oracle 분석 함수

**4. 기존 SQL 재사용:**
- 기존에 작성된 SQL 쿼리 활용
- 복잡한 뷰(VIEW) 조회

---

## 2. 기본 사용법

### 2.1 @Query 어노테이션 사용

**기본 문법:**

```java
@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {
    @Query(value = "SELECT * FROM orders WHERE status = ?1", nativeQuery = true)
    List<Order> findByStatus(String status);
}
```

**파라미터 바인딩:**

```java
// 위치 기반 파라미터 (?1, ?2, ...)
@Query(value = "SELECT * FROM orders WHERE status = ?1 AND user_id = ?2", 
       nativeQuery = true)
List<Order> findByStatusAndUserId(String status, Long userId);

// 이름 기반 파라미터 (:paramName)
@Query(value = "SELECT * FROM orders WHERE status = :status AND user_id = :userId", 
       nativeQuery = true)
List<Order> findByStatusAndUserIdNamed(@Param("status") String status, 
                                      @Param("userId") Long userId);
```

### 2.2 EntityManager 사용

**EntityManager로 직접 실행:**

```java
@Service
public class OrderService {
    @PersistenceContext
    private EntityManager entityManager;
    
    public List<Order> findOrdersNative(String status) {
        String sql = "SELECT * FROM orders WHERE status = :status";
        Query query = entityManager.createNativeQuery(sql, Order.class);
        query.setParameter("status", status);
        return query.getResultList();
    }
}
```

**Object[] 반환:**

```java
public List<Object[]> findOrderStats() {
    String sql = """
        SELECT 
            status,
            COUNT(*) as count,
            SUM(total_amount) as total
        FROM orders
        GROUP BY status
        """;
    Query query = entityManager.createNativeQuery(sql);
    return query.getResultList();
}
```

---

## 3. 반환 타입 처리

### 3.1 Entity 반환

**엔티티로 자동 매핑:**

```java
@Query(value = "SELECT * FROM orders WHERE status = ?1", nativeQuery = true)
List<Order> findByStatus(String status);
```

**주의사항:**
- SELECT 절에 모든 컬럼 포함 필요
- 컬럼명이 엔티티 필드명과 일치해야 함
- @Column(name = "...") 어노테이션 확인

### 3.2 Object[] 반환

**여러 컬럼 조회:**

```java
@Query(value = """
    SELECT o.id, o.date, o.status, u.name, u.email
    FROM orders o
    INNER JOIN users u ON o.user_id = u.id
    WHERE o.status = ?1
    """, nativeQuery = true)
List<Object[]> findOrdersWithUser(String status);
```

**사용 예시:**

```java
List<Object[]> results = orderRepository.findOrdersWithUser("PENDING");
for (Object[] row : results) {
    Long orderId = ((Number) row[0]).longValue();
    LocalDateTime date = (LocalDateTime) row[1];
    String status = (String) row[2];
    String userName = (String) row[3];
    String userEmail = (String) row[4];
}
```

**문제점:**
- 타입 안전성 부족
- 인덱스 기반 접근 (가독성 낮음)
- 런타임 에러 가능

### 3.3 DTO 매핑 (권장)

**인터페이스 기반 DTO:**

```java
public interface OrderSummaryDTO {
    Long getOrderId();
    LocalDateTime getOrderDate();
    String getStatus();
    String getUserName();
    String getUserEmail();
}

@Query(value = """
    SELECT 
        o.id as orderId,
        o.date as orderDate,
        o.status as status,
        u.name as userName,
        u.email as userEmail
    FROM orders o
    INNER JOIN users u ON o.user_id = u.id
    WHERE o.status = ?1
    """, nativeQuery = true)
List<OrderSummaryDTO> findOrderSummaries(String status);
```

**클래스 기반 DTO (SqlResultSetMapping):**

```java
@Entity
@SqlResultSetMapping(
    name = "OrderSummaryMapping",
    classes = @ConstructorResult(
        targetClass = OrderSummaryDTO.class,
        columns = {
            @ColumnResult(name = "orderId", type = Long.class),
            @ColumnResult(name = "orderDate", type = LocalDateTime.class),
            @ColumnResult(name = "status", type = String.class),
            @ColumnResult(name = "userName", type = String.class),
            @ColumnResult(name = "userEmail", type = String.class)
        }
    )
)
public class Order {
    // ...
}

public class OrderSummaryDTO {
    private Long orderId;
    private LocalDateTime orderDate;
    private String status;
    private String userName;
    private String userEmail;
    
    public OrderSummaryDTO(Long orderId, LocalDateTime orderDate, 
                          String status, String userName, String userEmail) {
        this.orderId = orderId;
        this.orderDate = orderDate;
        this.status = status;
        this.userName = userName;
        this.userEmail = userEmail;
    }
}

@Query(value = """
    SELECT 
        o.id as orderId,
        o.date as orderDate,
        o.status as status,
        u.name as userName,
        u.email as userEmail
    FROM orders o
    INNER JOIN users u ON o.user_id = u.id
    WHERE o.status = ?1
    """, 
    nativeQuery = true,
    resultSetMapping = "OrderSummaryMapping")
List<OrderSummaryDTO> findOrderSummaries(String status);
```

**Spring Data JPA Projection (가장 간단):**

```java
public interface OrderSummaryProjection {
    Long getOrderId();
    LocalDateTime getOrderDate();
    String getStatus();
    String getUserName();
    String getUserEmail();
}

@Query(value = """
    SELECT 
        o.id as orderId,
        o.date as orderDate,
        o.status as status,
        u.name as userName,
        u.email as userEmail
    FROM orders o
    INNER JOIN users u ON o.user_id = u.id
    WHERE o.status = ?1
    """, nativeQuery = true)
List<OrderSummaryProjection> findOrderSummaries(String status);
```

---

## 4. 페이징 처리

### 4.1 Pageable 사용

**Spring Data JPA 페이징:**

```java
@Query(value = """
    SELECT o.*, u.name, u.email
    FROM orders o
    INNER JOIN users u ON o.user_id = u.id
    WHERE o.status = ?1
    ORDER BY o.created_at DESC
    """,
    countQuery = "SELECT COUNT(*) FROM orders o WHERE o.status = ?1",
    nativeQuery = true)
Page<Order> findOrdersWithUser(String status, Pageable pageable);
```

**중요:**
- `countQuery`를 별도로 지정해야 함
- 페이징 쿼리와 카운트 쿼리를 분리하여 성능 최적화

### 4.2 수동 페이징 (LIMIT/OFFSET)

**직접 LIMIT/OFFSET 사용:**

```java
@Query(value = """
    SELECT o.*, u.name, u.email
    FROM orders o
    INNER JOIN users u ON o.user_id = u.id
    WHERE o.status = ?1
    ORDER BY o.created_at DESC
    LIMIT ?2 OFFSET ?3
    """, nativeQuery = true)
List<Order> findOrdersWithUserPaged(String status, int limit, int offset);
```

**사용 예시:**

```java
int page = 0;
int size = 20;
List<Order> orders = orderRepository.findOrdersWithUserPaged(
    "PENDING", 
    size, 
    page * size
);
```

---

## 5. 복잡한 쿼리 작성

### 5.1 CTE (Common Table Expression)

**WITH 절 사용:**

```java
@Query(value = """
    WITH monthly_stats AS (
        SELECT 
            DATE_TRUNC('month', created_at) as month,
            COUNT(*) as order_count,
            SUM(total_amount) as total_revenue
        FROM orders
        WHERE created_at >= ?1
        GROUP BY DATE_TRUNC('month', created_at)
    )
    SELECT 
        month,
        order_count,
        total_revenue,
        LAG(total_revenue) OVER (ORDER BY month) as prev_month_revenue
    FROM monthly_stats
    ORDER BY month DESC
    """, nativeQuery = true)
List<Object[]> findMonthlyStatsWithGrowth(LocalDateTime startDate);
```

### 5.2 윈도우 함수

**ROW_NUMBER, RANK 사용:**

```java
@Query(value = """
    SELECT 
        id,
        user_id,
        total_amount,
        ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY total_amount DESC) as rank
    FROM orders
    WHERE status = 'COMPLETED'
    """, nativeQuery = true)
List<Object[]> findTopOrdersByUser();
```

### 5.3 복잡한 서브쿼리

**서브쿼리 활용:**

```java
@Query(value = """
    SELECT 
        o.*,
        (SELECT COUNT(*) FROM order_items oi WHERE oi.order_id = o.id) as item_count,
        (SELECT SUM(oi.quantity * oi.price) FROM order_items oi WHERE oi.order_id = o.id) as total_item_amount
    FROM orders o
    WHERE o.status = ?1
    """, nativeQuery = true)
List<Object[]> findOrdersWithItemStats(String status);
```

---

## 6. DB 특화 기능 활용

### 6.1 PostgreSQL Full-Text Search

**텍스트 검색:**

```java
@Query(value = """
    SELECT o.*, 
           ts_rank(to_tsvector('english', o.description), 
                   plainto_tsquery('english', ?1)) as rank
    FROM orders o
    WHERE to_tsvector('english', o.description) @@ plainto_tsquery('english', ?1)
    ORDER BY rank DESC
    LIMIT 100
    """, nativeQuery = true)
List<Order> searchOrdersByDescription(String query);
```

### 6.2 MySQL JSON 함수

**JSON 데이터 처리:**

```java
@Query(value = """
    SELECT 
        id,
        JSON_EXTRACT(metadata, '$.source') as source,
        JSON_EXTRACT(metadata, '$.campaign') as campaign
    FROM orders
    WHERE JSON_EXTRACT(metadata, '$.source') = ?1
    """, nativeQuery = true)
List<Object[]> findOrdersByMetadata(String source);
```

### 6.3 인덱스 힌트

**MySQL 인덱스 힌트:**

```java
@Query(value = """
    SELECT /*+ USE_INDEX(orders, idx_status_created_at) */
           o.*, u.name
    FROM orders o
    INNER JOIN users u ON o.user_id = u.id
    WHERE o.status = ?1
      AND o.created_at >= ?2
    ORDER BY o.created_at DESC
    LIMIT 100
    """, nativeQuery = true)
List<Order> findRecentOrdersWithIndexHint(String status, LocalDateTime startDate);
```

**PostgreSQL 인덱스 힌트:**

```java
@Query(value = """
    SELECT /*+ INDEX(orders idx_status_created_at) */
           o.*, u.name
    FROM orders o
    INNER JOIN users u ON o.user_id = u.id
    WHERE o.status = ?1
      AND o.created_at >= ?2
    ORDER BY o.created_at DESC
    LIMIT 100
    """, nativeQuery = true)
List<Order> findRecentOrdersWithIndexHint(String status, LocalDateTime startDate);
```

---

## 7. 주의사항 및 Best Practices

### 7.1 SQL Injection 방지

**❌ 위험한 코드:**

```java
// ❌ SQL Injection 위험
@Query(value = "SELECT * FROM orders WHERE status = '" + status + "'", 
       nativeQuery = true)
List<Order> findByStatusUnsafe(String status);
```

**✅ 안전한 코드:**

```java
// ✅ 파라미터 바인딩 사용
@Query(value = "SELECT * FROM orders WHERE status = ?1", nativeQuery = true)
List<Order> findByStatus(String status);

// ✅ 이름 기반 파라미터
@Query(value = "SELECT * FROM orders WHERE status = :status", nativeQuery = true)
List<Order> findByStatusNamed(@Param("status") String status);
```

### 7.2 타입 안전성 확보

**❌ Object[] 사용 (타입 안전성 부족):**

```java
@Query(value = "SELECT id, date, status FROM orders", nativeQuery = true)
List<Object[]> findAllUnsafe();
```

**✅ DTO/Projection 사용 (타입 안전성 확보):**

```java
public interface OrderSummary {
    Long getId();
    LocalDateTime getDate();
    String getStatus();
}

@Query(value = """
    SELECT id, date, status 
    FROM orders
    """, nativeQuery = true)
List<OrderSummary> findAllSafe();
```

### 7.3 DB 벤더 독립성 관리

**문제:**
- 네이티브 쿼리는 DB 벤더별 SQL 문법 사용
- MySQL, PostgreSQL, Oracle 등 각각 다른 문법

**해결 방법 1: 프로파일별 분리**

```java
@Profile("mysql")
@Repository
public interface OrderRepositoryMySQL extends JpaRepository<Order, Long> {
    @Query(value = """
        SELECT * FROM orders 
        WHERE created_at >= DATE_SUB(NOW(), INTERVAL 1 DAY)
        """, nativeQuery = true)
    List<Order> findRecentOrders();
}

@Profile("postgresql")
@Repository
public interface OrderRepositoryPostgres extends JpaRepository<Order, Long> {
    @Query(value = """
        SELECT * FROM orders 
        WHERE created_at >= NOW() - INTERVAL '1 day'
        """, nativeQuery = true)
    List<Order> findRecentOrders();
}
```

**해결 방법 2: 추상화 레이어**

```java
public interface OrderRepositoryCustom {
    List<Order> findRecentOrders();
}

@Repository
public class OrderRepositoryImpl implements OrderRepositoryCustom {
    @PersistenceContext
    private EntityManager entityManager;
    
    @Value("${spring.jpa.database}")
    private String database;
    
    public List<Order> findRecentOrders() {
        String sql = switch (database) {
            case "mysql" -> """
                SELECT * FROM orders 
                WHERE created_at >= DATE_SUB(NOW(), INTERVAL 1 DAY)
                """;
            case "postgresql" -> """
                SELECT * FROM orders 
                WHERE created_at >= NOW() - INTERVAL '1 day'
                """;
            default -> throw new UnsupportedOperationException("Unsupported database: " + database);
        };
        
        return entityManager.createNativeQuery(sql, Order.class).getResultList();
    }
}
```

### 7.4 성능 최적화

**1. 필요한 컬럼만 선택:**

```java
// ❌ 모든 컬럼 조회
@Query(value = "SELECT * FROM orders", nativeQuery = true)

// ✅ 필요한 컬럼만 선택
@Query(value = "SELECT id, date, status FROM orders", nativeQuery = true)
```

**2. 인덱스 활용:**

```java
@Query(value = """
    SELECT /*+ USE_INDEX(orders, idx_status_created_at) */
           o.*
    FROM orders o
    WHERE o.status = ?1
      AND o.created_at >= ?2
    """, nativeQuery = true)
List<Order> findOrdersOptimized(String status, LocalDateTime startDate);
```

**3. COUNT 쿼리 최적화:**

```java
@Query(value = """
    SELECT o.*, u.name
    FROM orders o
    INNER JOIN users u ON o.user_id = u.id
    WHERE o.status = ?1
    ORDER BY o.created_at DESC
    """,
    countQuery = "SELECT COUNT(*) FROM orders o WHERE o.status = ?1",
    nativeQuery = true)
Page<Order> findOrdersWithUser(String status, Pageable pageable);
```

### 7.5 쿼리 가독성

**❌ 가독성 낮은 코드:**

```java
@Query(value = "SELECT o.*,u.name,u.email FROM orders o INNER JOIN users u ON o.user_id=u.id WHERE o.status=?1 ORDER BY o.created_at DESC LIMIT ?2", nativeQuery = true)
```

**✅ 가독성 좋은 코드 (텍스트 블록):**

```java
@Query(value = """
    SELECT o.*, u.name, u.email
    FROM orders o
    INNER JOIN users u ON o.user_id = u.id
    WHERE o.status = ?1
    ORDER BY o.created_at DESC
    LIMIT ?2
    """, nativeQuery = true)
List<Order> findOrdersWithUser(String status, int limit);
```

---

## 8. 실전 예시

### 8.1 복잡한 통계 쿼리

**월별 주문 통계 (CTE + 윈도우 함수):**

```java
public interface MonthlyOrderStats {
    LocalDateTime getMonth();
    Long getOrderCount();
    BigDecimal getTotalRevenue();
    BigDecimal getPrevMonthRevenue();
    Double getGrowthRate();
}

@Query(value = """
    WITH monthly_stats AS (
        SELECT 
            DATE_TRUNC('month', created_at) as month,
            COUNT(*) as order_count,
            SUM(total_amount) as total_revenue
        FROM orders
        WHERE created_at >= ?1
        GROUP BY DATE_TRUNC('month', created_at)
    )
    SELECT 
        month,
        order_count,
        total_revenue,
        LAG(total_revenue) OVER (ORDER BY month) as prev_month_revenue,
        CASE 
            WHEN LAG(total_revenue) OVER (ORDER BY month) > 0 
            THEN ((total_revenue - LAG(total_revenue) OVER (ORDER BY month)) / 
                  LAG(total_revenue) OVER (ORDER BY month)) * 100
            ELSE 0
        END as growth_rate
    FROM monthly_stats
    ORDER BY month DESC
    """, nativeQuery = true)
List<MonthlyOrderStats> findMonthlyStatsWithGrowth(LocalDateTime startDate);
```

### 8.2 대량 데이터 처리

**배치 조회 (커서 기반 페이징):**

```java
@Query(value = """
    SELECT o.*
    FROM orders o
    WHERE o.id > ?1
      AND o.status = ?2
    ORDER BY o.id ASC
    LIMIT ?3
    """, nativeQuery = true)
List<Order> findOrdersAfterId(Long lastId, String status, int batchSize);
```

**사용 예시:**

```java
public void processOrdersInBatches(String status, int batchSize) {
    Long lastId = 0L;
    List<Order> orders;
    
    do {
        orders = orderRepository.findOrdersAfterId(lastId, status, batchSize);
        processOrders(orders);
        if (!orders.isEmpty()) {
            lastId = orders.get(orders.size() - 1).getId();
        }
    } while (!orders.isEmpty());
}
```

### 8.3 복잡한 검색 쿼리

**다중 조건 검색:**

```java
public interface OrderSearchResult {
    Long getOrderId();
    LocalDateTime getOrderDate();
    String getStatus();
    String getUserName();
    Long getItemCount();
    BigDecimal getTotalAmount();
}

@Query(value = """
    SELECT 
        o.id as orderId,
        o.date as orderDate,
        o.status as status,
        u.name as userName,
        COUNT(oi.id) as itemCount,
        SUM(oi.quantity * oi.price) as totalAmount
    FROM orders o
    INNER JOIN users u ON o.user_id = u.id
    LEFT JOIN order_items oi ON o.id = oi.order_id
    WHERE 
        (:status IS NULL OR o.status = :status)
        AND (:startDate IS NULL OR o.created_at >= :startDate)
        AND (:endDate IS NULL OR o.created_at <= :endDate)
        AND (:userName IS NULL OR u.name LIKE CONCAT('%', :userName, '%'))
    GROUP BY o.id, o.date, o.status, u.name
    HAVING (:minAmount IS NULL OR SUM(oi.quantity * oi.price) >= :minAmount)
    ORDER BY o.created_at DESC
    LIMIT :limit OFFSET :offset
    """, nativeQuery = true)
List<OrderSearchResult> searchOrders(
    @Param("status") String status,
    @Param("startDate") LocalDateTime startDate,
    @Param("endDate") LocalDateTime endDate,
    @Param("userName") String userName,
    @Param("minAmount") BigDecimal minAmount,
    @Param("limit") int limit,
    @Param("offset") int offset
);
```

---

## 9. 디버깅 및 모니터링

### 9.1 쿼리 로그 확인

**application.yml 설정:**

```yaml
spring:
  jpa:
    properties:
      hibernate:
        format_sql: true
        show_sql: true
        use_sql_comments: true
```

**로그 출력 예시:**

```sql
-- Hibernate: 
    SELECT o.*, u.name, u.email
    FROM orders o
    INNER JOIN users u ON o.user_id = u.id
    WHERE o.status = ?
```

### 9.2 실행 계획 확인

**PostgreSQL:**

```sql
EXPLAIN ANALYZE
SELECT o.*, u.name, u.email
FROM orders o
INNER JOIN users u ON o.user_id = u.id
WHERE o.status = 'PENDING';
```

**MySQL:**

```sql
EXPLAIN
SELECT o.*, u.name, u.email
FROM orders o
INNER JOIN users u ON o.user_id = u.id
WHERE o.status = 'PENDING';
```

### 9.3 성능 모니터링

**쿼리 실행 시간 측정:**

```java
@Service
public class OrderService {
    private static final Logger log = LoggerFactory.getLogger(OrderService.class);
    
    public List<Order> findOrdersWithMonitoring(String status) {
        long startTime = System.currentTimeMillis();
        
        List<Order> orders = orderRepository.findByStatus(status);
        
        long executionTime = System.currentTimeMillis() - startTime;
        log.info("Query execution time: {}ms, Result count: {}", 
                 executionTime, orders.size());
        
        if (executionTime > 1000) {
            log.warn("Slow query detected: {}ms", executionTime);
        }
        
        return orders;
    }
}
```

---

## 마무리

**핵심 포인트:**

- **네이티브 쿼리는 복잡한 SQL을 직접 작성할 수 있는 강력한 기능입니다.**
- **SQL Injection 방지를 위해 반드시 파라미터 바인딩을 사용해야 합니다.**
- **타입 안전성을 확보하기 위해 DTO/Projection을 사용하는 것이 좋습니다.**
- **DB 벤더 독립성이 필요한 경우 프로파일별 분리 또는 추상화 레이어를 고려해야 합니다.**
- **성능 최적화를 위해 필요한 컬럼만 선택하고, 인덱스를 적절히 활용해야 합니다.**

**최종 권장사항:**

1. **복잡한 집계, 대량 데이터, DB 특화 기능이 필요한 경우에만 네이티브 쿼리 사용**
2. **파라미터 바인딩으로 SQL Injection 방지**
3. **DTO/Projection으로 타입 안전성 확보**
4. **쿼리 가독성을 위해 텍스트 블록 사용**
5. **성능 모니터링 및 실행 계획 확인**

네이티브 쿼리를 올바르게 사용하면 JPA의 제한사항을 우회하여 복잡한 쿼리를 효율적으로 처리할 수 있습니다. 하지만 SQL Injection, 타입 안전성, DB 벤더 독립성 등의 주의사항을 반드시 고려해야 합니다. 🚀



