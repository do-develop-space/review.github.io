---
layout: post
title: "역직렬화가 필요한 이유: 객체 직렬화의 이해"
date: 2025-11-28
categories: [programming]
tags: [직렬화, 역직렬화, Serialization, Deserialization, JSON, 네트워크통신]
---

# 역직렬화가 필요한 이유: 객체 직렬화의 이해

프로그래밍을 하다 보면 "직렬화(Serialization)"와 "역직렬화(Deserialization)"라는 용어를 자주 접하게 됩니다. 특히 네트워크 통신, 데이터베이스 저장, 캐싱 등을 구현할 때 필수적인 개념입니다. 이번 포스트에서는 직렬화와 역직렬화가 무엇인지, 왜 필요한지, 그리고 실제로 어떻게 사용되는지 알아보겠습니다.

## 직렬화와 역직렬화란?

### 직렬화 (Serialization)

**직렬화**는 객체를 **바이트 스트림이나 문자열 형태로 변환**하는 과정입니다. 메모리에 있는 객체를 네트워크로 전송하거나 파일에 저장할 수 있는 형태로 만드는 것입니다.

```
객체 (메모리) → 직렬화 → 바이트 스트림/문자열
```

### 역직렬화 (Deserialization)

**역직렬화**는 직렬화된 데이터를 다시 **원래의 객체로 복원**하는 과정입니다.

```
바이트 스트림/문자열 → 역직렬화 → 객체 (메모리)
```

## 왜 직렬화가 필요한가?

### 1. 네트워크 통신

가장 대표적인 사용 사례입니다. 서로 다른 프로세스나 서버 간에 객체를 전송할 때는 직렬화가 필수입니다.

**문제 상황:**
```java
// 서버 A
User user = new User("홍길동", "hong@example.com");
// 이 객체를 서버 B로 전송하려면?
```

**메모리 주소는 전송할 수 없습니다!** 서버 A의 메모리 주소를 서버 B로 보내도 의미가 없습니다. 따라서 객체를 전송 가능한 형태(바이트 스트림, JSON 등)로 변환해야 합니다.

**해결:**
```java
// 서버 A: 직렬화
String json = objectMapper.writeValueAsString(user);
// {"name":"홍길동","email":"hong@example.com"}

// 네트워크 전송
sendOverNetwork(json);

// 서버 B: 역직렬화
User receivedUser = objectMapper.readValue(json, User.class);
```

### 2. 데이터베이스 저장

객체를 데이터베이스에 저장할 때도 직렬화가 필요합니다.

**예시:**
```java
@Entity
public class Order {
    @Id
    private Long id;
    
    // 복잡한 객체를 JSON으로 저장
    @Column(columnDefinition = "TEXT")
    private String shippingAddress; // JSON 문자열로 저장
    
    // 또는 JPA Converter 사용
    @Convert(converter = AddressConverter.class)
    private Address address; // 자동으로 직렬화/역직렬화
}
```

### 3. 캐싱

Redis 같은 캐시 시스템에 객체를 저장할 때도 직렬화가 필요합니다.

```java
// 캐시에 저장 (직렬화)
User user = userService.findById(1L);
redisTemplate.opsForValue().set("user:1", user);
// 내부적으로 직렬화되어 저장됨

// 캐시에서 조회 (역직렬화)
User cachedUser = redisTemplate.opsForValue().get("user:1");
// 내부적으로 역직렬화되어 객체로 복원됨
```

### 4. 메시지 큐

Kafka, RabbitMQ 같은 메시지 브로커에 객체를 전송할 때도 직렬화가 필요합니다.

```java
// 메시지 전송 (직렬화)
OrderEvent event = new OrderEvent(orderId, userId, amount);
kafkaTemplate.send("order-events", event);
// 내부적으로 직렬화되어 전송됨

// 메시지 수신 (역직렬화)
@KafkaListener(topics = "order-events")
public void handleOrderEvent(OrderEvent event) {
    // 역직렬화된 객체 사용
}
```

## 직렬화 방식

### 1. JSON (가장 일반적)

**장점:**
- 사람이 읽을 수 있음
- 언어 독립적
- 웹 표준

**단점:**
- 바이너리보다 크기가 큼
- 타입 정보 손실 (역직렬화 시 타입 지정 필요)

```java
// Jackson 사용 예제
ObjectMapper mapper = new ObjectMapper();

// 직렬화
User user = new User("홍길동", "hong@example.com");
String json = mapper.writeValueAsString(user);
// {"name":"홍길동","email":"hong@example.com"}

// 역직렬화
User deserialized = mapper.readValue(json, User.class);
```

### 2. Java 직렬화

**장점:**
- Java 네이티브
- 타입 정보 보존
- 간단한 구현

**단점:**
- Java에 종속적
- 보안 취약점
- 성능 문제

```java
// 직렬화
User user = new User("홍길동", "hong@example.com");
ByteArrayOutputStream baos = new ByteArrayOutputStream();
ObjectOutputStream oos = new ObjectOutputStream(baos);
oos.writeObject(user);
byte[] serialized = baos.toByteArray();

// 역직렬화
ByteArrayInputStream bais = new ByteArrayInputStream(serialized);
ObjectInputStream ois = new ObjectInputStream(bais);
User deserialized = (User) ois.readObject();
```

### 3. Protocol Buffers (Protobuf)

**장점:**
- 매우 작은 크기
- 빠른 성능
- 언어 독립적
- 스키마 버전 관리

**단점:**
- 스키마 정의 필요
- 사람이 읽을 수 없음

```java
// Protobuf 정의
message User {
  string name = 1;
  string email = 2;
}

// 직렬화
UserProto.User user = UserProto.User.newBuilder()
    .setName("홍길동")
    .setEmail("hong@example.com")
    .build();
byte[] serialized = user.toByteArray();

// 역직렬화
UserProto.User deserialized = UserProto.User.parseFrom(serialized);
```

## 역직렬화가 실패하는 경우

### 1. 타입 불일치

```java
// 직렬화: User 객체
String json = "{\"name\":\"홍길동\",\"email\":\"hong@example.com\"}";

// 역직렬화: 잘못된 타입
Product product = mapper.readValue(json, Product.class);
// ❌ 실패: 필드가 맞지 않음
```

### 2. 스키마 변경

```java
// 이전 버전: email 필드만 있음
String oldJson = "{\"email\":\"hong@example.com\"}";

// 새 버전: name 필드 추가됨
public class User {
    private String name;  // 새로 추가된 필드
    private String email;
}

// 역직렬화 시 name은 null
User user = mapper.readValue(oldJson, User.class);
// ✅ 성공하지만 name은 null
```

### 3. 버전 호환성

```java
// 서버 A: v1.0
public class User {
    private String name;
    private String email;
}

// 서버 B: v2.0 (필드 추가)
public class User {
    private String name;
    private String email;
    private String phone;  // 새 필드
}

// v1.0에서 직렬화된 데이터를 v2.0에서 역직렬화
// ✅ phone은 null이지만 성공
```

## 실제 사용 예제

### REST API에서의 사용

```java
@RestController
public class UserController {
    
    // 요청 본문 역직렬화
    @PostMapping("/users")
    public ResponseEntity<User> createUser(@RequestBody User user) {
        // JSON → User 객체 (역직렬화)
        User created = userService.create(user);
        return ResponseEntity.ok(created);
        // User 객체 → JSON (직렬화)
    }
    
    // 응답 본문 직렬화
    @GetMapping("/users/{id}")
    public ResponseEntity<User> getUser(@PathVariable Long id) {
        User user = userService.findById(id);
        return ResponseEntity.ok(user);
        // User 객체 → JSON (직렬화)
    }
}
```

### 메시지 큐에서의 사용

```java
// Kafka Producer (직렬화)
@Service
public class OrderEventProducer {
    
    @Autowired
    private KafkaTemplate<String, OrderEvent> kafkaTemplate;
    
    public void publishOrderCreated(Order order) {
        OrderEvent event = new OrderEvent(
            order.getId(),
            order.getUserId(),
            order.getAmount()
        );
        // OrderEvent → JSON (직렬화)
        kafkaTemplate.send("order-events", event);
    }
}

// Kafka Consumer (역직렬화)
@Component
public class OrderEventConsumer {
    
    @KafkaListener(topics = "order-events")
    public void handleOrderEvent(OrderEvent event) {
        // JSON → OrderEvent (역직렬화)
        // 이미 Spring이 자동으로 역직렬화해줌
        processOrder(event);
    }
}
```

### Redis 캐싱에서의 사용

```java
@Service
public class UserCacheService {
    
    @Autowired
    private RedisTemplate<String, User> redisTemplate;
    
    public void cacheUser(User user) {
        // User → 직렬화 → Redis 저장
        redisTemplate.opsForValue().set("user:" + user.getId(), user);
    }
    
    public User getCachedUser(Long id) {
        // Redis 조회 → 역직렬화 → User
        return redisTemplate.opsForValue().get("user:" + id);
    }
}
```

## 직렬화/역직렬화 성능 고려사항

### 1. JSON vs Protobuf

```
JSON:
  - 크기: 100 bytes
  - 직렬화 시간: 1ms
  - 역직렬화 시간: 1ms

Protobuf:
  - 크기: 50 bytes (50% 작음)
  - 직렬화 시간: 0.3ms (3배 빠름)
  - 역직렬화 시간: 0.3ms (3배 빠름)
```

### 2. 직렬화 최적화

```java
// 느린 방법: 매번 ObjectMapper 생성
ObjectMapper mapper = new ObjectMapper();
String json = mapper.writeValueAsString(user);

// 빠른 방법: 재사용
private static final ObjectMapper mapper = new ObjectMapper();
String json = mapper.writeValueAsString(user);
```

## 보안 고려사항

### 1. 신뢰할 수 없는 데이터 역직렬화 금지

```java
// ❌ 위험: 사용자 입력을 직접 역직렬화
String userInput = request.getParameter("data");
User user = mapper.readValue(userInput, User.class);
// 악의적인 데이터로 인한 보안 취약점

// ✅ 안전: 검증 후 역직렬화
String userInput = request.getParameter("data");
if (isValidJson(userInput)) {
    User user = mapper.readValue(userInput, User.class);
}
```

### 2. Java 직렬화의 위험성

Java의 기본 직렬화는 보안 취약점이 있어 사용을 지양해야 합니다:

```java
// ❌ 위험: Java 직렬화
ObjectInputStream ois = new ObjectInputStream(input);
Object obj = ois.readObject();  // 임의 코드 실행 가능

// ✅ 안전: JSON 사용
ObjectMapper mapper = new ObjectMapper();
User user = mapper.readValue(json, User.class);
```

## 결론

직렬화와 역직렬화는 현대 소프트웨어 개발에서 필수적인 개념입니다:

### 직렬화가 필요한 이유

1. **네트워크 통신**: 객체를 다른 프로세스/서버로 전송
2. **데이터 저장**: 객체를 데이터베이스나 파일에 저장
3. **캐싱**: 객체를 캐시 시스템에 저장
4. **메시지 큐**: 객체를 메시지 브로커에 전송

### 역직렬화가 필요한 이유

1. **데이터 복원**: 직렬화된 데이터를 다시 객체로 변환
2. **타입 안정성**: 타입 정보를 유지하여 안전하게 사용
3. **객체 조작**: 역직렬화된 객체를 프로그램에서 조작

### 선택 가이드

- **일반적인 경우**: JSON (가장 널리 사용, 호환성 좋음)
- **고성능 필요**: Protocol Buffers (작은 크기, 빠른 속도)
- **Java 내부**: Java 직렬화 (하지만 보안 위험으로 지양)

**Yellow Store 프로젝트에서는 REST API와 메시지 큐에서 JSON을 사용하여 직렬화/역직렬화를 수행하고 있습니다.** JSON은 사람이 읽을 수 있고, 언어 독립적이며, 웹 표준이기 때문에 마이크로서비스 아키텍처에 적합합니다.

직렬화와 역직렬화를 올바르게 이해하고 사용하면, 분산 시스템에서 안전하고 효율적인 데이터 전송과 저장이 가능합니다. 🚀

---

다음 포스트에서는 **Rate Limiting과 Access Token 탈취 방어**에 대해 다루겠습니다. 대량의 데이터 요청을 방지하는 접근 제한 서버의 동작 원리와, Access Token이 탈취되어도 안전한 이유, 그리고 실제 구현 방법에 대한 보안 가이드를 제공하겠습니다. 많은 관심 부탁드립니다! 🔒

