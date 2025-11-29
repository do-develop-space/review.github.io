---
layout: post
title: "Rate Limiting과 Access Token 탈취 방어: 보안 가이드"
date: 2025-11-29
categories: [security]
tags: [Rate Limiting, 보안, Access Token, DDoS 방어, API 보안, Redis]
---

# Rate Limiting과 Access Token 탈취 방어: 보안 가이드

API를 개발하다 보면 "대량의 데이터 요청을 방지하는 접근 제한 서버"라는 개념을 접하게 됩니다. 이것은 **Rate Limiting(레이트 리미팅)** 또는 **Throttling(스로틀링)**이라고 불립니다. 

특히 중요한 점은, **Access Token이 탈취되어도 Rate Limiting을 통해 추가적인 보안 계층을 제공**할 수 있다는 것입니다. 이번 포스트에서는 Rate Limiting의 개념, Access Token 탈취 시에도 안전한 이유, 그리고 실제 구현 방법에 대해 알아보겠습니다.

## Rate Limiting이란?

Rate Limiting은 **일정 시간 동안 허용되는 요청 수를 제한**하는 보안 메커니즘입니다. 이를 통해 다음과 같은 목적을 달성할 수 있습니다:

- **DDoS 공격 방어**: 대량의 요청으로 인한 서버 과부하 방지
- **리소스 보호**: 서버 자원을 합리적으로 사용하도록 제한
- **비용 관리**: API 사용량에 따른 비용 제어
- **공정한 사용**: 모든 사용자가 공정하게 API를 사용할 수 있도록 보장

### 기본 동작 원리

```
사용자 → API 요청 → Rate Limiter → 허용/차단 결정 → 서버
                              ↓
                    [요청 카운트 확인]
                    - 시간당 100회 제한
                    - 초당 10회 제한
```

## Access Token 탈취 시에도 안전한 이유

### 문제 상황: Access Token 탈취

Access Token이 탈취되면 공격자는 다음과 같은 공격을 시도할 수 있습니다:

1. **대량의 요청**: 탈취한 토큰으로 무제한 API 호출
2. **데이터 수집**: 대량의 데이터를 빠르게 수집
3. **서버 과부하**: 서버를 다운시키는 공격
4. **비용 증가**: API 사용량 증가로 인한 비용 폭증

### Rate Limiting의 보호 메커니즘

Rate Limiting은 **토큰이 탈취되어도 추가적인 보안 계층**을 제공합니다:

#### 1. 요청 빈도 제한

```
정상 사용자: 1분에 10회 요청 → ✅ 허용
공격자 (탈취 토큰): 1분에 10,000회 요청 → ❌ 차단
```

**예시:**
```java
// Rate Limiter 설정: 사용자당 분당 100회 제한
if (requestCount > 100) {
    return ResponseEntity.status(429).body("Too Many Requests");
}
```

#### 2. IP 기반 추가 제한

토큰과 함께 **IP 주소도 함께 확인**하여 이중 보호:

```java
// 토큰 + IP 조합으로 제한
String key = token + ":" + clientIp;
if (getRequestCount(key) > limit) {
    return ResponseEntity.status(429).body("Rate limit exceeded");
}
```

#### 3. 사용 패턴 분석

정상적인 사용 패턴과 다른 요청을 감지:

```
정상: 사용자가 1시간에 걸쳐 100회 요청
공격: 탈취 토큰으로 1분에 100회 요청 → 의심스러운 패턴
```

#### 4. 점진적 제한

의심스러운 활동 감지 시 점진적으로 제한 강화:

```java
// 1단계: 경고 (429 Too Many Requests)
// 2단계: 짧은 차단 (1분)
// 3단계: 긴 차단 (1시간)
// 4단계: 토큰 무효화
```

## Rate Limiting 구현 방법

### 1. Redis를 이용한 Rate Limiting

Redis는 Rate Limiting 구현에 가장 널리 사용되는 도구입니다. **Sliding Window** 또는 **Fixed Window** 알고리즘을 사용합니다.

#### Sliding Window 알고리즘

```java
@Service
public class RateLimitingService {
    
    @Autowired
    private RedisTemplate<String, String> redisTemplate;
    
    public boolean isAllowed(String key, int maxRequests, int windowSeconds) {
        String redisKey = "rate_limit:" + key;
        String current = redisTemplate.opsForValue().get(redisKey);
        
        if (current == null) {
            // 첫 요청: 카운터 초기화
            redisTemplate.opsForValue().set(
                redisKey, "1", 
                Duration.ofSeconds(windowSeconds)
            );
            return true;
        }
        
        int count = Integer.parseInt(current);
        if (count >= maxRequests) {
            // 제한 초과
            return false;
        }
        
        // 카운터 증가
        redisTemplate.opsForValue().increment(redisKey);
        return true;
    }
}
```

#### Fixed Window 알고리즘

```java
public boolean isAllowedFixedWindow(String key, int maxRequests, int windowSeconds) {
    long currentWindow = System.currentTimeMillis() / (windowSeconds * 1000);
    String redisKey = "rate_limit:" + key + ":" + currentWindow;
    
    String count = redisTemplate.opsForValue().get(redisKey);
    if (count == null) {
        redisTemplate.opsForValue().set(
            redisKey, "1",
            Duration.ofSeconds(windowSeconds)
        );
        return true;
    }
    
    int requestCount = Integer.parseInt(count);
    if (requestCount >= maxRequests) {
        return false;
    }
    
    redisTemplate.opsForValue().increment(redisKey);
    return true;
}
```

### 2. Spring Cloud Gateway에서의 Rate Limiting

Spring Cloud Gateway는 내장된 Rate Limiting 기능을 제공합니다.

#### 의존성 추가

```gradle
dependencies {
    implementation 'org.springframework.cloud:spring-cloud-starter-gateway'
    implementation 'org.springframework.boot:spring-boot-starter-data-redis-reactive'
    implementation 'com.github.vladimir-bukhtoyarov:bucket4j-core:7.6.0'
}
```

#### Gateway 설정

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: lb://user-service
          predicates:
            - Path=/api/users/**
          filters:
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 10  # 초당 10개 토큰 추가
                redis-rate-limiter.burstCapacity: 20  # 최대 20개 토큰
                redis-rate-limiter.requestedTokens: 1 # 요청당 1개 토큰
                key-resolver: "#{@userKeyResolver}"     # 키 리졸버
```

#### 키 리졸버 구현

```java
@Configuration
public class RateLimitingConfig {
    
    @Bean
    public KeyResolver userKeyResolver() {
        return exchange -> {
            // 사용자 ID 또는 토큰 기반 키 생성
            String token = exchange.getRequest()
                .getHeaders()
                .getFirst("Authorization");
            
            if (token != null) {
                // 토큰에서 사용자 ID 추출
                String userId = extractUserIdFromToken(token);
                return Mono.just(userId);
            }
            
            // IP 주소 기반
            String ip = exchange.getRequest()
                .getRemoteAddress()
                .getAddress()
                .getHostAddress();
            return Mono.just(ip);
        };
    }
}
```

### 3. 다층 Rate Limiting 전략

보안을 강화하기 위해 **여러 계층에서 Rate Limiting**을 적용합니다:

#### 계층 1: IP 기반 제한

```java
@Component
public class IpBasedRateLimiter {
    
    public boolean isAllowed(String ip) {
        // IP당 초당 100회 제한
        return rateLimitingService.isAllowed(
            "ip:" + ip, 
            100, 
            1
        );
    }
}
```

#### 계층 2: 사용자 기반 제한

```java
@Component
public class UserBasedRateLimiter {
    
    public boolean isAllowed(String userId) {
        // 사용자당 분당 1000회 제한
        return rateLimitingService.isAllowed(
            "user:" + userId,
            1000,
            60
        );
    }
}
```

#### 계층 3: 토큰 기반 제한

```java
@Component
public class TokenBasedRateLimiter {
    
    public boolean isAllowed(String token) {
        // 토큰당 시간당 10000회 제한
        return rateLimitingService.isAllowed(
            "token:" + token,
            10000,
            3600
        );
    }
}
```

#### 통합 사용

```java
@RestController
public class ApiController {
    
    @Autowired
    private IpBasedRateLimiter ipLimiter;
    
    @Autowired
    private UserBasedRateLimiter userLimiter;
    
    @Autowired
    private TokenBasedRateLimiter tokenLimiter;
    
    @GetMapping("/api/data")
    public ResponseEntity<?> getData(
            @RequestHeader("Authorization") String token,
            HttpServletRequest request) {
        
        String ip = request.getRemoteAddr();
        String userId = extractUserId(token);
        
        // 다층 검증
        if (!ipLimiter.isAllowed(ip)) {
            return ResponseEntity.status(429)
                .body("IP rate limit exceeded");
        }
        
        if (!userLimiter.isAllowed(userId)) {
            return ResponseEntity.status(429)
                .body("User rate limit exceeded");
        }
        
        if (!tokenLimiter.isAllowed(token)) {
            return ResponseEntity.status(429)
                .body("Token rate limit exceeded");
        }
        
        // 모든 검증 통과 시 데이터 반환
        return ResponseEntity.ok(getData());
    }
}
```

## 고급 보안 전략

### 1. 동적 Rate Limiting

사용자의 행동 패턴에 따라 Rate Limit을 동적으로 조정:

```java
@Service
public class DynamicRateLimitingService {
    
    public int getRateLimit(String userId) {
        UserBehavior behavior = analyzeUserBehavior(userId);
        
        if (behavior.isSuspicious()) {
            // 의심스러운 사용자: 제한 강화
            return 10;  // 분당 10회
        } else if (behavior.isPremium()) {
            // 프리미엄 사용자: 제한 완화
            return 1000;  // 분당 1000회
        } else {
            // 일반 사용자: 기본 제한
            return 100;  // 분당 100회
        }
    }
}
```

### 2. 지리적 제한

특정 지역에서의 요청을 제한:

```java
@Service
public class GeoBasedRateLimiting {
    
    public boolean isAllowed(String ip, String country) {
        // 특정 국가에서의 요청 제한
        if (isRestrictedCountry(country)) {
            return rateLimitingService.isAllowed(
                "geo:" + country,
                50,  // 분당 50회
                60
            );
        }
        return true;
    }
}
```

### 3. 엔드포인트별 제한

API 엔드포인트마다 다른 제한 적용:

```java
@Configuration
public class EndpointRateLimitingConfig {
    
    private Map<String, RateLimitConfig> endpointLimits = Map.of(
        "/api/users", new RateLimitConfig(100, 60),      // 분당 100회
        "/api/orders", new RateLimitConfig(50, 60),      // 분당 50회
        "/api/payments", new RateLimitConfig(20, 60)     // 분당 20회
    );
    
    public RateLimitConfig getLimit(String endpoint) {
        return endpointLimits.getOrDefault(
            endpoint, 
            new RateLimitConfig(100, 60)  // 기본값
        );
    }
}
```

### 4. 토큰 탈취 감지 및 대응

의심스러운 활동 감지 시 자동 대응:

```java
@Service
public class TokenTheftDetectionService {
    
    public void detectSuspiciousActivity(String token, String ip) {
        // 1. 일반적인 사용 위치와 다른 IP에서 요청
        String normalIp = getNormalIpForToken(token);
        if (!normalIp.equals(ip)) {
            log.warn("Suspicious IP change for token: {}", token);
            // 추가 검증 요구
        }
        
        // 2. 비정상적인 요청 패턴
        int requestCount = getRecentRequestCount(token);
        if (requestCount > 1000) {  // 1시간에 1000회 이상
            log.warn("Abnormal request pattern detected: {}", token);
            // 토큰 일시 정지 또는 무효화
            suspendToken(token);
        }
        
        // 3. 여러 IP에서 동시 사용
        Set<String> activeIps = getActiveIpsForToken(token);
        if (activeIps.size() > 3) {  // 3개 이상의 IP에서 동시 사용
            log.warn("Multiple IP usage detected: {}", token);
            // 토큰 무효화 및 사용자에게 알림
            invalidateToken(token);
            notifyUser(token);
        }
    }
}
```

## 실제 구현 예제

### 완전한 Rate Limiting 시스템

```java
@Aspect
@Component
public class RateLimitingAspect {
    
    @Autowired
    private RateLimitingService rateLimitingService;
    
    @Autowired
    private TokenTheftDetectionService theftDetection;
    
    @Around("@annotation(RateLimited)")
    public Object rateLimit(ProceedingJoinPoint joinPoint) throws Throwable {
        HttpServletRequest request = getRequest(joinPoint);
        String token = extractToken(request);
        String ip = request.getRemoteAddr();
        String userId = extractUserId(token);
        String endpoint = request.getRequestURI();
        
        // 1. IP 기반 제한
        if (!rateLimitingService.isAllowed("ip:" + ip, 100, 60)) {
            throw new RateLimitExceededException("IP rate limit exceeded");
        }
        
        // 2. 사용자 기반 제한
        if (!rateLimitingService.isAllowed("user:" + userId, 1000, 60)) {
            throw new RateLimitExceededException("User rate limit exceeded");
        }
        
        // 3. 토큰 기반 제한
        if (!rateLimitingService.isAllowed("token:" + token, 10000, 3600)) {
            throw new RateLimitExceededException("Token rate limit exceeded");
        }
        
        // 4. 엔드포인트별 제한
        RateLimitConfig config = getEndpointLimit(endpoint);
        if (!rateLimitingService.isAllowed(
                "endpoint:" + endpoint + ":" + userId,
                config.getMaxRequests(),
                config.getWindowSeconds())) {
            throw new RateLimitExceededException("Endpoint rate limit exceeded");
        }
        
        // 5. 토큰 탈취 감지
        theftDetection.detectSuspiciousActivity(token, ip);
        
        // 모든 검증 통과 시 요청 처리
        return joinPoint.proceed();
    }
}
```

### 커스텀 어노테이션

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface RateLimited {
    int maxRequests() default 100;
    int windowSeconds() default 60;
    RateLimitType type() default RateLimitType.USER;
}

public enum RateLimitType {
    IP,      // IP 기반
    USER,    // 사용자 기반
    TOKEN,   // 토큰 기반
    ENDPOINT // 엔드포인트 기반
}
```

### 사용 예제

```java
@RestController
public class DataController {
    
    @RateLimited(maxRequests = 100, windowSeconds = 60, type = RateLimitType.USER)
    @GetMapping("/api/data")
    public ResponseEntity<List<Data>> getData() {
        return ResponseEntity.ok(dataService.getAllData());
    }
    
    @RateLimited(maxRequests = 10, windowSeconds = 60, type = RateLimitType.ENDPOINT)
    @PostMapping("/api/expensive-operation")
    public ResponseEntity<?> expensiveOperation() {
        // 비용이 많이 드는 작업
        return ResponseEntity.ok(expensiveService.process());
    }
}
```

## 보안 모니터링 및 알림

### 의심스러운 활동 로깅

```java
@Service
public class SecurityMonitoringService {
    
    public void logSuspiciousActivity(String token, String ip, String reason) {
        SecurityEvent event = SecurityEvent.builder()
            .token(token)
            .ip(ip)
            .reason(reason)
            .timestamp(LocalDateTime.now())
            .severity(Severity.HIGH)
            .build();
        
        // 로그 저장
        securityLogRepository.save(event);
        
        // 알림 전송
        if (event.getSeverity() == Severity.CRITICAL) {
            alertService.sendCriticalAlert(event);
        }
    }
}
```

### Rate Limit 초과 시 응답

```java
@ControllerAdvice
public class RateLimitExceptionHandler {
    
    @ExceptionHandler(RateLimitExceededException.class)
    public ResponseEntity<ErrorResponse> handleRateLimit(
            RateLimitExceededException e,
            HttpServletRequest request) {
        
        // Retry-After 헤더 추가
        HttpHeaders headers = new HttpHeaders();
        headers.add("Retry-After", "60");  // 60초 후 재시도
        
        ErrorResponse error = ErrorResponse.builder()
            .code("RATE_LIMIT_EXCEEDED")
            .message("Too many requests. Please try again later.")
            .retryAfter(60)
            .build();
        
        return ResponseEntity
            .status(HttpStatus.TOO_MANY_REQUESTS)
            .headers(headers)
            .body(error);
    }
}
```

## 모범 사례

### 1. 다층 방어

- **네트워크 레벨**: Nginx, CloudFlare 등에서 기본 제한
- **게이트웨이 레벨**: API Gateway에서 Rate Limiting
- **애플리케이션 레벨**: 애플리케이션 내부에서 세밀한 제어

### 2. 점진적 제한

```java
// 1단계: 경고
if (count > limit * 0.8) {
    addWarningHeader(response);
}

// 2단계: 제한
if (count > limit) {
    return 429 Too Many Requests;
}

// 3단계: 차단
if (count > limit * 2) {
    blockIp(ip);
}
```

### 3. 사용자 경험 고려

```java
// Rate Limit 정보를 헤더로 제공
response.setHeader("X-RateLimit-Limit", "100");
response.setHeader("X-RateLimit-Remaining", "50");
response.setHeader("X-RateLimit-Reset", "1638360000");
```

### 4. 정기적인 모니터링

- Rate Limit 위반 패턴 분석
- 의심스러운 활동 자동 감지
- 정기적인 보안 리포트 생성

## 결론

Rate Limiting은 **Access Token이 탈취되어도 추가적인 보안 계층을 제공**하는 중요한 메커니즘입니다:

### 주요 보호 메커니즘

1. **요청 빈도 제한**: 대량 요청 차단
2. **다층 검증**: IP, 사용자, 토큰 기반 제한
3. **패턴 분석**: 비정상적인 사용 패턴 감지
4. **자동 대응**: 의심스러운 활동 시 자동 차단

### 구현 권장사항

- ✅ **Redis 기반**: 빠르고 확장 가능한 Rate Limiting
- ✅ **다층 방어**: 여러 계층에서 Rate Limiting 적용
- ✅ **동적 조정**: 사용자 행동에 따른 동적 제한
- ✅ **모니터링**: 지속적인 보안 모니터링 및 알림

**Yellow Store 프로젝트에서는 Redis를 활용한 다층 Rate Limiting을 공부하고 있으며, 적용할 예정입니다.** 이를 통해 DDoS 공격 방어, 리소스 보호, 그리고 공정한 API 사용을 보장할 수 있습니다.

보안은 한 번에 완성되는 것이 아니라 지속적으로 개선해야 하는 과정입니다. Rate Limiting은 그 과정에서 중요한 한 부분입니다. 🔒

---

다음 포스트에서는 **IaaS, PaaS, SaaS 개념 정리와 메모리 관리 전략**에 대해 다루겠습니다. 클라우드 서비스 모델별 특징과, 애플리케이션 메모리 관리 관점에서 어떤 차이가 있는지 정리해보겠습니다. 피드백 환영합니다! 🧠

