---
layout: post
title: "JPA vs TypeORM vs Django ORM: N+1 문제 해결 방식 비교"
date: 2026-01-19
categories: [programming, database, orm]
tags: [JPA, TypeORM, DjangoORM, N+1문제, ORM비교, 성능최적화, FetchJoin, select_related, prefetch_related, EagerLoading, LazyLoading]
---

이전 글에서 Redis 캐시 스탬피드 문제를 다뤘습니다. 이번 글에서는 **다양한 ORM에서 N+1 문제를 해결하는 방식의 차이점**을 비교해보겠습니다.

JPA(Java), TypeORM(TypeScript/Node.js), Django ORM(Python)은 각각 다른 철학과 접근 방식으로 N+1 문제를 해결합니다. 각 ORM의 특징과 해결 방법을 비교하여 실제 프로젝트에서 선택할 때 도움이 되도록 정리하겠습니다.

---

## 1. N+1 문제란? (공통 개념)

### 1.1 문제 정의

**N+1 문제**는 모든 ORM에서 공통적으로 발생하는 성능 문제입니다:

1. **1번의 쿼리**로 N개의 엔티티를 조회
2. 각 엔티티의 연관 관계를 조회하기 위해 **추가로 N번의 쿼리** 실행
3. 총 **1 + N번의 쿼리**가 실행됨

### 1.2 공통 예시 모델

**엔티티 관계:**
- `Order` (주문) - `User` (사용자): ManyToOne
- `User` (사용자) - `Order` (주문): OneToMany

**문제 발생 코드 (공통 패턴):**
```java
// 1번의 쿼리: 모든 주문 조회
List<Order> orders = orderRepository.findAll();

// N번의 쿼리: 각 주문의 사용자 조회
for (Order order : orders) {
    User user = order.getUser();  // 각 주문마다 쿼리 실행!
}
```

---

## 2. JPA (Java Persistence API) - Hibernate

### 2.1 특징

- **프록시 기반 Lazy Loading**: Hibernate Proxy 객체 사용
- **영속성 컨텍스트**: 1차 캐시로 같은 엔티티 재조회 방지
- **JPQL (Java Persistence Query Language)**: 객체 지향 쿼리 언어

### 2.2 해결 방법

#### 방법 1: Fetch Join (가장 권장)

**JPQL 사용:**
```java
@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {
    @Query("SELECT o FROM Order o JOIN FETCH o.user")
    List<Order> findAllWithUser();
}
```

**생성되는 SQL:**
```sql
SELECT o.*, u.* 
FROM orders o 
INNER JOIN users u ON o.user_id = u.user_id;
```

**장점:**
- 한 번의 쿼리로 모든 데이터 조회
- 페치 조인은 가장 효율적

**단점:**
- 페이징과 함께 사용 불가 (예외 발생)
- 여러 컬렉션 동시 Fetch Join 시 카테시안 곱 문제

#### 방법 2: @EntityGraph

**메서드 이름 기반 쿼리에서도 사용 가능:**
```java
@Entity
@NamedEntityGraph(
    name = "Order.withUser",
    attributeNodes = @NamedAttributeNode("user")
)
public class Order {
    // ...
}

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

#### 방법 3: @BatchSize

**배치로 연관 관계 조회:**
```java
@Entity
@BatchSize(size = 10)
public class User {
    @OneToMany(mappedBy = "user")
    private List<Order> orders;
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

**장점:**
- 페이징과 함께 사용 가능
- N번의 쿼리를 N/batchSize번으로 줄임

**단점:**
- 완전한 해결은 아님 (여전히 여러 번의 쿼리 실행)

#### 방법 4: DTO Projection

**필요한 데이터만 조회:**
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

---

## 3. TypeORM (TypeScript/Node.js)

### 3.1 특징

- **Active Record 패턴** 또는 **Data Mapper 패턴** 지원
- **Relation Decorator**: `@ManyToOne`, `@OneToMany` 등으로 관계 정의
- **QueryBuilder**: 유연한 쿼리 작성

### 3.2 해결 방법

#### 방법 1: Relations 옵션 (Eager Loading)

**엔티티 정의 시:**
```typescript
import { Entity, PrimaryGeneratedColumn, Column, ManyToOne, JoinColumn } from 'typeorm';

@Entity()
export class Order {
    @PrimaryGeneratedColumn()
    id: number;
    
    @ManyToOne(() => User, user => user.orders, { eager: true })  // Eager Loading
    @JoinColumn({ name: 'user_id' })
    user: User;
}
```

**사용:**
```typescript
const orders = await orderRepository.find();  // User도 함께 조회됨
```

**단점:**
- 항상 함께 조회되어 불필요한 데이터까지 가져올 수 있음
- 순환 참조 문제 가능

#### 방법 2: Relations 옵션 (find 메서드)

**find 메서드의 relations 옵션:**
```typescript
const orders = await orderRepository.find({
    relations: ['user']  // User도 함께 조회
});
```

**생성되는 SQL:**
```sql
SELECT o.*, u.* 
FROM orders o 
LEFT JOIN users u ON o.user_id = u.user_id;
```

**장점:**
- 필요할 때만 연관 관계 로딩
- 페이징과 함께 사용 가능

**단점:**
- LEFT JOIN 사용 (INNER JOIN이 더 효율적일 수 있음)

#### 방법 3: QueryBuilder (JOIN)

**명시적 JOIN:**
```typescript
const orders = await orderRepository
    .createQueryBuilder('order')
    .leftJoinAndSelect('order.user', 'user')  // JOIN FETCH와 유사
    .getMany();
```

**생성되는 SQL:**
```sql
SELECT o.*, u.* 
FROM orders o 
LEFT JOIN users u ON o.user_id = u.user_id;
```

**장점:**
- 복잡한 쿼리 작성 가능
- WHERE, ORDER BY 등과 함께 사용 가능

**단점:**
- 코드가 길어짐

#### 방법 4: DataLoader 패턴 (GraphQL 스타일)

**배치 로딩:**
```typescript
import DataLoader from 'dataloader';

// DataLoader 생성
const userLoader = new DataLoader<number, User>(async (userIds) => {
    const users = await userRepository.findByIds(userIds);
    const userMap = new Map(users.map(user => [user.id, user]));
    return userIds.map(id => userMap.get(id));
});

// 사용
const orders = await orderRepository.find();
const userIds = orders.map(order => order.userId);
const users = await userLoader.loadMany(userIds);

// 메모리에서 매핑
orders.forEach(order => {
    order.user = users.find(u => u.id === order.userId);
});
```

**장점:**
- GraphQL 환경에서 특히 유용
- 배치로 쿼리 최적화

**단점:**
- 추가 라이브러리 필요
- 구현이 복잡함

---

## 4. Django ORM (Python)

### 4.1 특징

- **Lazy Evaluation**: 쿼리셋은 실제 사용될 때까지 실행되지 않음
- **select_related**: ForeignKey, OneToOne 관계 최적화 (SQL JOIN)
- **prefetch_related**: ManyToMany, 역방향 ForeignKey 최적화 (별도 쿼리 + Python 조인)

### 4.2 해결 방법

#### 방법 1: select_related (ForeignKey, OneToOne)

**단일 쿼리로 JOIN:**
```python
from django.db import models

class Order(models.Model):
    user = models.ForeignKey(User, on_delete=models.CASCADE)
    order_date = models.DateTimeField()

# N+1 문제 발생
orders = Order.objects.all()
for order in orders:
    print(order.user.name)  # 각 주문마다 쿼리 실행

# 해결: select_related
orders = Order.objects.select_related('user').all()
for order in orders:
    print(order.user.name)  # 추가 쿼리 없음
```

**생성되는 SQL:**
```sql
SELECT o.*, u.* 
FROM orders o 
LEFT OUTER JOIN users u ON o.user_id = u.id;
```

**장점:**
- 한 번의 쿼리로 모든 데이터 조회
- Django ORM의 가장 효율적인 방법

**단점:**
- ForeignKey, OneToOne 관계만 지원
- ManyToMany, 역방향 ForeignKey에는 사용 불가

#### 방법 2: prefetch_related (ManyToMany, 역방향 ForeignKey)

**별도 쿼리 + Python 조인:**
```python
class User(models.Model):
    name = models.CharField(max_length=100)

class Order(models.Model):
    user = models.ForeignKey(User, on_delete=models.CASCADE, related_name='orders')
    order_date = models.DateTimeField()

# N+1 문제 발생
users = User.objects.all()
for user in users:
    print(user.orders.all())  # 각 사용자마다 쿼리 실행

# 해결: prefetch_related
users = User.objects.prefetch_related('orders').all()
for user in users:
    print(user.orders.all())  # 추가 쿼리 없음 (이미 메모리에 로드됨)
```

**생성되는 SQL:**
```sql
-- 첫 번째 쿼리: 사용자 조회
SELECT * FROM users;

-- 두 번째 쿼리: 모든 주문 조회 (IN 절 사용)
SELECT * FROM orders WHERE user_id IN (1, 2, 3, ...);
```

**동작 방식:**
1. 첫 번째 쿼리로 모든 User 조회
2. 두 번째 쿼리로 관련된 모든 Order 조회 (IN 절)
3. Python에서 메모리에서 조인

**장점:**
- ManyToMany, 역방향 ForeignKey에서도 사용 가능
- 2번의 쿼리로 해결 (N+1 → 2)

**단점:**
- select_related보다 쿼리 수가 많음 (1번 vs 2번)
- 메모리 사용량 증가 가능

#### 방법 3: select_related + prefetch_related 조합

**중첩 관계 최적화:**
```python
# Order -> User -> Profile
orders = Order.objects.select_related('user').prefetch_related('user__profile').all()
```

**생성되는 SQL:**
```sql
-- 1. Order와 User JOIN
SELECT o.*, u.* FROM orders o LEFT OUTER JOIN users u ON o.user_id = u.id;

-- 2. Profile 조회 (IN 절)
SELECT * FROM profiles WHERE user_id IN (1, 2, 3, ...);
```

#### 방법 4: Prefetch 객체 (고급 최적화)

**쿼리셋 커스터마이징:**
```python
from django.db.models import Prefetch

# 활성화된 주문만 prefetch
users = User.objects.prefetch_related(
    Prefetch('orders', queryset=Order.objects.filter(status='ACTIVE'))
).all()
```

**장점:**
- prefetch할 데이터를 필터링 가능
- 더 세밀한 제어

---

## 5. 비교표

### 5.1 해결 방법 비교

| ORM | 주요 해결 방법 | 쿼리 수 | 페이징 지원 | 특징 |
|-----|--------------|---------|------------|------|
| **JPA** | Fetch Join | 1번 | ❌ | JPQL 사용, 가장 효율적 |
| **JPA** | @EntityGraph | 1번 | ✅ (제한적) | 메서드 이름 기반 쿼리 지원 |
| **JPA** | @BatchSize | 1 + N/batchSize | ✅ | 배치로 쿼리 수 감소 |
| **TypeORM** | relations 옵션 | 1번 | ✅ | find 메서드에 옵션 추가 |
| **TypeORM** | QueryBuilder | 1번 | ✅ | 명시적 JOIN |
| **Django** | select_related | 1번 | ✅ | ForeignKey, OneToOne만 |
| **Django** | prefetch_related | 2번 | ✅ | ManyToMany, 역방향 FK |

### 5.2 철학적 차이

#### JPA (Hibernate)
- **프록시 기반**: Lazy Loading이 기본
- **명시적 최적화 필요**: Fetch Join, @EntityGraph 등으로 명시적으로 최적화
- **유연성**: 다양한 최적화 방법 제공

#### TypeORM
- **옵션 기반**: relations 옵션으로 간단하게 해결
- **QueryBuilder**: 복잡한 쿼리도 유연하게 작성
- **GraphQL 친화적**: DataLoader 패턴 지원

#### Django ORM
- **명확한 구분**: select_related vs prefetch_related
- **관계 타입별 최적화**: ForeignKey는 JOIN, ManyToMany는 별도 쿼리
- **Pythonic**: 간단하고 직관적인 API

---

## 6. 실제 사용 예시 비교

### 6.1 시나리오: 주문 목록과 사용자 정보 조회

#### JPA
```java
// 방법 1: Fetch Join
@Query("SELECT o FROM Order o JOIN FETCH o.user")
List<Order> findAllWithUser();

// 방법 2: @EntityGraph
@EntityGraph(attributePaths = {"user"})
List<Order> findAll();
```

#### TypeORM
```typescript
// 방법 1: relations 옵션
const orders = await orderRepository.find({
    relations: ['user']
});

// 방법 2: QueryBuilder
const orders = await orderRepository
    .createQueryBuilder('order')
    .leftJoinAndSelect('order.user', 'user')
    .getMany();
```

#### Django ORM
```python
# select_related 사용
orders = Order.objects.select_related('user').all()
```

### 6.2 시나리오: 사용자 목록과 주문 목록 조회 (OneToMany)

#### JPA
```java
// 방법 1: Fetch Join (DISTINCT 필요)
@Query("SELECT DISTINCT u FROM User u JOIN FETCH u.orders")
List<User> findAllWithOrders();

// 방법 2: @BatchSize
@Entity
@BatchSize(size = 10)
public class User {
    @OneToMany(mappedBy = "user")
    private List<Order> orders;
}
```

#### TypeORM
```typescript
// relations 옵션
const users = await userRepository.find({
    relations: ['orders']
});

// QueryBuilder
const users = await userRepository
    .createQueryBuilder('user')
    .leftJoinAndSelect('user.orders', 'order')
    .getMany();
```

#### Django ORM
```python
# prefetch_related 사용 (역방향 ForeignKey)
users = User.objects.prefetch_related('orders').all()
```

---

## 7. 각 ORM의 장단점

### 7.1 JPA (Hibernate)

**장점:**
- 가장 성숙한 ORM (오랜 기간 사용)
- 다양한 최적화 방법 제공
- 엔터프라이즈 환경에서 검증됨

**단점:**
- 학습 곡선이 가파름
- Fetch Join과 페이징 호환성 문제
- 설정이 복잡할 수 있음

### 7.2 TypeORM

**장점:**
- TypeScript 타입 안정성
- 간단한 API (relations 옵션)
- GraphQL과 잘 통합됨

**단점:**
- 상대적으로 최근에 등장 (덜 성숙)
- 일부 고급 기능 부족
- 성능 최적화 옵션이 제한적

### 7.3 Django ORM

**장점:**
- 매우 직관적인 API
- select_related/prefetch_related 명확한 구분
- Pythonic하고 읽기 쉬움

**단점:**
- Python에 종속적
- 복잡한 쿼리 작성 시 제한적
- prefetch_related는 메모리 사용량 증가

---

## 8. 선택 가이드

### 8.1 프로젝트 타입별 추천

**엔터프라이즈 Java 프로젝트:**
- **JPA (Hibernate)** 추천
- 검증된 안정성과 다양한 최적화 방법

**TypeScript/Node.js 프로젝트:**
- **TypeORM** 추천
- TypeScript 타입 안정성과 간단한 API

**Python/Django 프로젝트:**
- **Django ORM** 추천
- Django와 완벽한 통합

### 8.2 성능이 중요한 경우

**최고 성능:**
- **JPA Fetch Join** (1번의 쿼리)
- **Django select_related** (1번의 쿼리)

**페이징 필요:**
- **JPA @BatchSize** (N/batchSize번의 쿼리)
- **TypeORM relations** (1번의 쿼리, LEFT JOIN)
- **Django prefetch_related** (2번의 쿼리)

### 8.3 개발 생산성

**가장 간단:**
- **Django select_related/prefetch_related** (명확한 구분)
- **TypeORM relations 옵션** (간단한 API)

**가장 유연:**
- **JPA QueryBuilder** (복잡한 쿼리 작성 가능)
- **TypeORM QueryBuilder** (유연한 쿼리 작성)

---

## 9. Best Practice

### 9.1 공통 Best Practice

1. **쿼리 로그 확인**: 항상 실제 실행되는 쿼리 확인
2. **프로파일링**: 성능 측정 도구 사용
3. **인덱스 최적화**: 외래 키에 인덱스 생성
4. **필요한 데이터만 조회**: DTO Projection 활용

### 9.2 JPA Best Practice

```java
// ✅ DO: Fetch Join 사용
@Query("SELECT o FROM Order o JOIN FETCH o.user")
List<Order> findAllWithUser();

// ✅ DO: 페이징 시 @BatchSize
@BatchSize(size = 10)

// ❌ DON'T: Eager Loading 남용
@ManyToOne(fetch = FetchType.EAGER)  // 피하세요
```

### 9.3 TypeORM Best Practice

```typescript
// ✅ DO: relations 옵션 사용
const orders = await orderRepository.find({
    relations: ['user']
});

// ✅ DO: QueryBuilder로 복잡한 쿼리 작성
const orders = await orderRepository
    .createQueryBuilder('order')
    .leftJoinAndSelect('order.user', 'user')
    .where('order.status = :status', { status: 'ACTIVE' })
    .getMany();

// ❌ DON'T: Eager Loading 남용
@ManyToOne(() => User, { eager: true })  // 피하세요
```

### 9.4 Django ORM Best Practice

```python
# ✅ DO: select_related (ForeignKey, OneToOne)
orders = Order.objects.select_related('user').all()

# ✅ DO: prefetch_related (ManyToMany, 역방향 FK)
users = User.objects.prefetch_related('orders').all()

# ✅ DO: 중첩 관계 최적화
orders = Order.objects.select_related('user').prefetch_related('user__profile').all()

# ❌ DON'T: N+1 문제 무시
orders = Order.objects.all()  # 피하세요
for order in orders:
    print(order.user.name)  # N+1 문제 발생
```

---

## 10. 모니터링 및 디버깅

### 10.1 JPA

**쿼리 로그 활성화:**
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
```

### 10.2 TypeORM

**쿼리 로그 활성화:**
```typescript
import { createConnection } from 'typeorm';

createConnection({
    type: 'postgres',
    // ...
    logging: true,  // 모든 쿼리 로그
    logger: 'advanced-console'  // 상세 로그
});
```

### 10.3 Django ORM

**쿼리 로그 활성화:**
```python
# settings.py
LOGGING = {
    'version': 1,
    'handlers': {
        'console': {
            'class': 'logging.StreamHandler',
        },
    },
    'loggers': {
        'django.db.backends': {
            'level': 'DEBUG',
        },
    },
}

# 또는 django-debug-toolbar 사용
```

**쿼리 개수 확인:**
```python
from django.db import connection

print(len(connection.queries))  # 실행된 쿼리 개수
```

---

## 마무리

**핵심 포인트:**

- **JPA**: Fetch Join이 가장 효율적이지만 페이징과 호환성 문제. @BatchSize로 보완
- **TypeORM**: relations 옵션이 간단하지만 LEFT JOIN 사용. QueryBuilder로 세밀한 제어 가능
- **Django ORM**: select_related (JOIN)와 prefetch_related (별도 쿼리) 명확한 구분

각 ORM은 서로 다른 철학과 접근 방식을 가지고 있지만, **N+1 문제를 해결하는 핵심은 동일합니다**: 연관 관계를 미리 로딩하여 추가 쿼리를 방지하는 것입니다.

프로젝트의 요구사항과 팀의 경험에 따라 적절한 ORM을 선택하고, 각 ORM의 최적화 방법을 올바르게 활용하는 것이 중요합니다. 🚀

다음 글에서는 **Spring Boot와 NestJS의 DDD 프로젝트 구조 설계 차이점**을 비교해보겠습니다.
