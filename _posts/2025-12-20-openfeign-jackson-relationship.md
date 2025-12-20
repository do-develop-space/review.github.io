---
layout: post
title: "OpenFeign과 Jackson의 연관 관계: HTTP 통신에서의 JSON 직렬화"
date: 2025-12-20
categories: [programming, microservices]
tags: [OpenFeign, Jackson, JSON, 직렬화, 역직렬화, HTTP클라이언트, 마이크로서비스]
---

# OpenFeign과 Jackson의 연관 관계: HTTP 통신에서의 JSON 직렬화

마이크로서비스 아키텍처에서 서비스 간 통신을 위해 **OpenFeign**을 사용할 때, 요청과 응답 데이터는 **JSON 형식**으로 직렬화/역직렬화됩니다.  
이 과정에서 **Jackson**이 핵심적인 역할을 담당합니다.

이 글에서는 OpenFeign과 Jackson이 어떻게 연동되는지, 그리고 실제로 어떤 과정을 거쳐 데이터가 변환되는지 정리해보겠습니다.

---

## 1. OpenFeign이란?

### OpenFeign의 개념

**OpenFeign**은 Netflix에서 개발한 **선언적 HTTP 클라이언트**입니다.  
인터페이스만 정의하면 자동으로 HTTP 요청을 생성하고 처리합니다.

**주요 특징:**
- 인터페이스 기반 선언적 프로그래밍
- Spring Cloud와 통합
- 자동 JSON 직렬화/역직렬화
- 로드 밸런싱 지원 (Ribbon, Spring Cloud LoadBalancer)

### 기본 사용 예시

```java
@FeignClient(name = "user-service", url = "http://localhost:8080")
public interface UserClient {
    @GetMapping("/api/users/{id}")
    UserResponse getUser(@PathVariable Long id);
    
    @PostMapping("/api/users")
    UserResponse createUser(@RequestBody UserRequest request);
}
```

**사용하는 쪽:**
```java
@Service
public class UserService {
    private final UserClient userClient;
    
    public UserResponse getUser(Long id) {
        return userClient.getUser(id);  // HTTP GET 요청 자동 생성
    }
}
```

---

## 2. Jackson이란?

### Jackson의 개념

**Jackson**은 Java 객체와 JSON 간의 **직렬화(Serialization)**와 **역직렬화(Deserialization)**를 처리하는 라이브러리입니다.

**주요 기능:**
- 객체 → JSON 변환 (직렬화)
- JSON → 객체 변환 (역직렬화)
- 어노테이션 기반 커스터마이징
- 다양한 데이터 타입 지원

### 기본 사용 예시

```java
ObjectMapper objectMapper = new ObjectMapper();

// 객체 → JSON (직렬화)
User user = new User("John", "john@example.com");
String json = objectMapper.writeValueAsString(user);
// {"name":"John","email":"john@example.com"}

// JSON → 객체 (역직렬화)
String json = "{\"name\":\"John\",\"email\":\"john@example.com\"}";
User user = objectMapper.readValue(json, User.class);
```

---

## 3. OpenFeign과 Jackson의 연관 관계

### 연관 관계 개요

**OpenFeign과 Jackson의 관계:**
1. **OpenFeign**: HTTP 요청/응답을 처리하는 클라이언트
2. **Jackson**: 요청/응답 데이터를 JSON으로 변환하는 직렬화/역직렬화 엔진
3. **Spring**: OpenFeign과 Jackson을 통합하여 자동으로 처리

**데이터 흐름:**
```
Java 객체 (UserRequest)
  ↓
Jackson 직렬화
  ↓
JSON 문자열 ("{\"name\":\"John\"}")
  ↓
HTTP 요청 본문
  ↓
OpenFeign이 HTTP 요청 전송
  ↓
HTTP 응답 본문 (JSON 문자열)
  ↓
Jackson 역직렬화
  ↓
Java 객체 (UserResponse)
```

### 실제 동작 과정

**1. 요청 전송 과정:**

```java
@FeignClient(name = "user-service")
public interface UserClient {
    @PostMapping("/api/users")
    UserResponse createUser(@RequestBody UserRequest request);
}

// 사용
UserRequest request = new UserRequest("John", "john@example.com");
UserResponse response = userClient.createUser(request);
```

**내부 동작:**
```
1. UserRequest 객체 생성
   UserRequest request = new UserRequest("John", "john@example.com");

2. Jackson이 객체를 JSON으로 직렬화
   String json = objectMapper.writeValueAsString(request);
   // {"name":"John","email":"john@example.com"}

3. OpenFeign이 HTTP POST 요청 생성
   POST /api/users
   Content-Type: application/json
   Body: {"name":"John","email":"john@example.com"}

4. HTTP 요청 전송
```

**2. 응답 수신 과정:**

```
1. HTTP 응답 수신
   HTTP/1.1 200 OK
   Content-Type: application/json
   Body: {"id":1,"name":"John","email":"john@example.com"}

2. Jackson이 JSON을 객체로 역직렬화
   UserResponse response = objectMapper.readValue(json, UserResponse.class);

3. UserResponse 객체 반환
   return response;
```

---

## 4. Spring에서의 통합

### 자동 설정

**Spring Boot는 OpenFeign과 Jackson을 자동으로 통합합니다:**

```java
@SpringBootApplication
@EnableFeignClients
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

**자동 설정 내용:**
1. **ObjectMapper 빈 등록**: Jackson의 ObjectMapper가 자동으로 등록됨
2. **HttpMessageConverter 등록**: Jackson을 사용하는 HttpMessageConverter 등록
3. **OpenFeign Encoder/Decoder 설정**: Jackson을 사용하는 Encoder/Decoder 자동 설정

### HttpMessageConverter

**Spring의 HttpMessageConverter:**
- HTTP 요청/응답의 본문을 변환하는 인터페이스
- Jackson을 사용하는 구현체: `MappingJackson2HttpMessageConverter`

**동작 과정:**
```java
// 요청 처리
@RequestBody UserRequest request
  ↓
MappingJackson2HttpMessageConverter
  ↓
Jackson ObjectMapper
  ↓
JSON → UserRequest 객체

// 응답 처리
UserResponse response
  ↓
MappingJackson2HttpMessageConverter
  ↓
Jackson ObjectMapper
  ↓
UserResponse 객체 → JSON
```

---

## 5. 실제 예시

### 예시 1: 기본 사용

**DTO 클래스:**
```java
public class UserRequest {
    private String name;
    private String email;
    
    // Jackson이 JSON 필드와 매핑
    // getter/setter 필요
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
}

public class UserResponse {
    private Long id;
    private String name;
    private String email;
    
    // getter/setter
}
```

**Feign Client:**
```java
@FeignClient(name = "user-service")
public interface UserClient {
    @PostMapping("/api/users")
    UserResponse createUser(@RequestBody UserRequest request);
}
```

**동작:**
```java
UserRequest request = new UserRequest("John", "john@example.com");
UserResponse response = userClient.createUser(request);

// 내부적으로:
// 1. Jackson이 UserRequest → JSON 변환
// 2. OpenFeign이 HTTP POST 요청 전송
// 3. Jackson이 JSON → UserResponse 변환
```

### 예시 2: Jackson 어노테이션 사용

**커스텀 필드명 매핑:**
```java
public class UserRequest {
    @JsonProperty("user_name")  // JSON 필드명 커스터마이징
    private String name;
    
    @JsonProperty("user_email")
    private String email;
    
    @JsonIgnore  // JSON 직렬화에서 제외
    private String internalField;
}
```

**JSON 결과:**
```json
{
  "user_name": "John",
  "user_email": "john@example.com"
  // internalField는 포함되지 않음
}
```

### 예시 3: 날짜 형식 커스터마이징

**날짜 필드:**
```java
public class UserResponse {
    private Long id;
    private String name;
    
    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss", timezone = "Asia/Seoul")
    private LocalDateTime createdAt;
}
```

**JSON 결과:**
```json
{
  "id": 1,
  "name": "John",
  "createdAt": "2025-12-20 10:30:00"
}
```

### 예시 4: 중첩 객체 처리

**중첩 객체:**
```java
public class OrderResponse {
    private Long orderId;
    private UserInfo user;  // 중첩 객체
    
    public static class UserInfo {
        private Long userId;
        private String userName;
        // getter/setter
    }
}
```

**JSON 결과:**
```json
{
  "orderId": 1,
  "user": {
    "userId": 100,
    "userName": "John"
  }
}
```

---

## 6. 커스터마이징

### ObjectMapper 커스터마이징

**전역 ObjectMapper 설정:**
```java
@Configuration
public class JacksonConfig {
    @Bean
    @Primary
    public ObjectMapper objectMapper() {
        ObjectMapper mapper = new ObjectMapper();
        mapper.setPropertyNamingStrategy(PropertyNamingStrategies.SNAKE_CASE);
        mapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
        mapper.registerModule(new JavaTimeModule());
        return mapper;
    }
}
```

**설정 내용:**
- `SNAKE_CASE`: 필드명을 snake_case로 변환
- `FAIL_ON_UNKNOWN_PROPERTIES = false`: 알 수 없는 속성 무시
- `JavaTimeModule`: LocalDateTime 등 Java 8 시간 타입 지원

### Feign Client별 커스터마이징

**특정 Feign Client에만 다른 설정 적용:**
```java
@FeignClient(
    name = "user-service",
    configuration = CustomFeignConfig.class
)
public interface UserClient {
    // ...
}

@Configuration
public class CustomFeignConfig {
    @Bean
    public Encoder feignEncoder() {
        ObjectMapper mapper = new ObjectMapper();
        mapper.setPropertyNamingStrategy(PropertyNamingStrategies.SNAKE_CASE);
        return new JacksonEncoder(mapper);
    }
    
    @Bean
    public Decoder feignDecoder() {
        ObjectMapper mapper = new ObjectMapper();
        mapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
        return new JacksonDecoder(mapper);
    }
}
```

---

## 7. 자주 발생하는 문제와 해결

### 문제 1: 직렬화 실패

**문제:**
```java
public class UserRequest {
    private String name;
    // getter/setter 없음
}

// 직렬화 실패: No serializer found
```

**해결:**
```java
// 방법 1: getter/setter 추가
public class UserRequest {
    private String name;
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
}

// 방법 2: @Data 사용 (Lombok)
@Data
public class UserRequest {
    private String name;
}

// 방법 3: @JsonAutoDetect 사용
@JsonAutoDetect(fieldVisibility = JsonAutoDetect.Visibility.ANY)
public class UserRequest {
    private String name;
}
```

### 문제 2: 역직렬화 실패

**문제:**
```java
// JSON에 없는 필드가 있으면 역직렬화 실패
public class UserResponse {
    private Long id;
    private String name;
    private String unknownField;  // JSON에 없음
}
```

**해결:**
```java
// 방법 1: @JsonIgnoreProperties 사용
@JsonIgnoreProperties(ignoreUnknown = true)
public class UserResponse {
    private Long id;
    private String name;
    private String unknownField;
}

// 방법 2: ObjectMapper 설정
objectMapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
```

### 문제 3: 날짜 형식 문제

**문제:**
```java
public class UserResponse {
    private LocalDateTime createdAt;  // 기본 형식으로 직렬화됨
}
```

**해결:**
```java
public class UserResponse {
    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss", timezone = "Asia/Seoul")
    private LocalDateTime createdAt;
}
```

### 문제 4: null 값 처리

**문제:**
```java
// null 값이 JSON에 포함되지 않음
public class UserRequest {
    private String name;  // null이면 JSON에서 제외
}
```

**해결:**
```java
// 방법 1: @JsonInclude 사용
@JsonInclude(JsonInclude.Include.ALWAYS)
public class UserRequest {
    private String name;  // null이어도 JSON에 포함
}

// 방법 2: ObjectMapper 설정
objectMapper.setSerializationInclusion(JsonInclude.Include.ALWAYS);
```

---

## 8. 성능 최적화

### ObjectMapper 재사용

**문제:**
```java
// 매번 새로운 ObjectMapper 생성 (비효율적)
ObjectMapper mapper = new ObjectMapper();
String json = mapper.writeValueAsString(user);
```

**해결:**
```java
// 빈으로 등록하여 재사용
@Bean
public ObjectMapper objectMapper() {
    return new ObjectMapper();
}

@Autowired
private ObjectMapper objectMapper;
```

### 직렬화 캐싱

**Jackson은 자동으로 직렬화 메타데이터를 캐싱:**
- 첫 번째 직렬화 시 메타데이터 생성 및 캐싱
- 이후 직렬화는 캐시된 메타데이터 사용
- 성능 최적화 자동 처리

---

## 9. 디버깅 팁

### 요청/응답 로깅

**Feign 로깅 활성화:**
```yaml
# application.yml
logging:
  level:
    com.example.client.UserClient: DEBUG
```

**로깅 설정:**
```java
@Configuration
public class FeignConfig {
    @Bean
    public Logger.Level feignLoggerLevel() {
        return Logger.Level.FULL;  // 요청/응답 전체 로깅
    }
}
```

**로그 출력:**
```
UserClient#createUser(UserRequest)
---> POST http://user-service/api/users
Content-Type: application/json
{"name":"John","email":"john@example.com"}
---> END HTTP
<--- HTTP/1.1 200 OK
{"id":1,"name":"John","email":"john@example.com"}
<--- END HTTP
```

### JSON 직렬화/역직렬화 확인

**수동 테스트:**
```java
@SpringBootTest
class JacksonTest {
    @Autowired
    private ObjectMapper objectMapper;
    
    @Test
    void testSerialization() throws Exception {
        UserRequest request = new UserRequest("John", "john@example.com");
        String json = objectMapper.writeValueAsString(request);
        System.out.println(json);
        // {"name":"John","email":"john@example.com"}
    }
    
    @Test
    void testDeserialization() throws Exception {
        String json = "{\"name\":\"John\",\"email\":\"john@example.com\"}";
        UserRequest request = objectMapper.readValue(json, UserRequest.class);
        System.out.println(request);
    }
}
```

---

## 마무리

**핵심 포인트:**

- **OpenFeign**: 선언적 HTTP 클라이언트, 인터페이스만으로 HTTP 요청 처리
- **Jackson**: JSON 직렬화/역직렬화 엔진, Java 객체와 JSON 간 변환
- **연관 관계**: OpenFeign이 HTTP 통신을 담당하고, Jackson이 데이터 변환을 담당
- **Spring 통합**: HttpMessageConverter를 통해 자동으로 통합되어 동작

**데이터 흐름:**
```
Java 객체 → Jackson 직렬화 → JSON → OpenFeign HTTP 요청
OpenFeign HTTP 응답 → JSON → Jackson 역직렬화 → Java 객체
```

OpenFeign과 Jackson은 마이크로서비스 간 통신에서 **불가분의 관계**입니다. OpenFeign이 HTTP 통신의 편의성을 제공하고, Jackson이 데이터 변환의 복잡성을 숨겨줍니다. **Jackson의 어노테이션과 설정을 이해**하면, 더 유연하고 안정적인 마이크로서비스 통신을 구현할 수 있습니다. 🚀

다음 글에서는 OpenFeign의 **에러 처리와 재시도 전략**에 대해 정리해보겠습니다.


