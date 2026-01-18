---
layout: post
title: "JWT 대칭키 vs 비대칭키: 어떤 방식을 선택해야 할까?"
date: 2025-11-26
categories: [programming]
tags: [JWT, 대칭키, 비대칭키, RSA, HMAC, 보안, 인증]
---

# JWT 대칭키 vs 비대칭키: 어떤 방식을 선택해야 할까?

JWT(JSON Web Token)를 사용할 때 가장 중요한 결정 중 하나는 서명 알고리즘을 선택하는 것입니다. 대칭키 방식(HMAC)과 비대칭키 방식(RSA) 중 어떤 것을 선택해야 할까요? 이번 포스트에서는 두 방식의 차이점, 장단점, 그리고 실제 사용 시나리오를 통해 올바른 선택 방법을 알아보겠습니다.

## JWT 서명 알고리즘 개요

JWT는 토큰의 무결성을 보장하기 위해 서명(Signature)을 사용합니다. 서명 알고리즘은 크게 두 가지로 나뉩니다:

1. **대칭키 방식 (Symmetric Key)**: HMAC-SHA256, HMAC-SHA512
2. **비대칭키 방식 (Asymmetric Key)**: RS256, ES256

## 대칭키 방식 (HMAC)

### 동작 원리

대칭키 방식은 **하나의 비밀키(Secret Key)**를 사용하여 토큰을 서명하고 검증합니다.

```
서명 생성: HMAC-SHA256(header.payload, secret_key)
검증: HMAC-SHA256(header.payload, secret_key) == signature
```

### 특징

**장점:**
- ✅ **빠른 성능**: 대칭키 암호화는 비대칭키보다 훨씬 빠름
- ✅ **간단한 구현**: 단일 키만 관리하면 됨
- ✅ **낮은 리소스 사용**: CPU와 메모리 사용량이 적음
- ✅ **확장성**: 대량의 토큰 처리에 유리

**단점:**
- ❌ **키 관리의 어려움**: 같은 키를 발급자와 검증자가 모두 알고 있어야 함
- ❌ **키 유출 위험**: 한 곳에서 키가 유출되면 전체 시스템이 위험
- ❌ **분산 환경에서의 제약**: 여러 서비스가 같은 키를 공유해야 함

### 구현 예제

```java
// 대칭키로 JWT 생성
String secretKey = "my-secret-key-256-bits-long";
Algorithm algorithm = Algorithm.HMAC256(secretKey);

String token = JWT.create()
    .withSubject("user123")
    .withExpiresAt(new Date(System.currentTimeMillis() + 3600000))
    .sign(algorithm);

// 대칭키로 JWT 검증
JWTVerifier verifier = JWT.require(algorithm).build();
DecodedJWT decoded = verifier.verify(token);
```

## 비대칭키 방식 (RSA)

### 동작 원리

비대칭키 방식은 **공개키(Public Key)와 개인키(Private Key) 쌍**을 사용합니다.

```
서명 생성: RS256(header.payload, private_key)
검증: RS256(header.payload, public_key) == signature
```

### 특징

**장점:**
- ✅ **보안성**: 개인키는 발급자만 가지고 있으면 됨
- ✅ **키 분리**: 공개키는 공개해도 안전함
- ✅ **분산 환경 적합**: 여러 서비스가 공개키만으로 검증 가능
- ✅ **키 회전 용이**: 개인키만 교체하면 됨

**단점:**
- ❌ **느린 성능**: 비대칭키 암호화는 대칭키보다 느림
- ❌ **높은 리소스 사용**: CPU 사용량이 많음
- ❌ **복잡한 구현**: 키 쌍 생성 및 관리가 복잡
- ❌ **확장성 제약**: 대량의 토큰 처리 시 성능 저하

### 구현 예제

```java
// 개인키로 JWT 생성
RSAPrivateKey privateKey = getPrivateKey();
Algorithm algorithm = Algorithm.RSA256(null, privateKey);

String token = JWT.create()
    .withSubject("user123")
    .withExpiresAt(new Date(System.currentTimeMillis() + 3600000))
    .sign(algorithm);

// 공개키로 JWT 검증
RSAPublicKey publicKey = getPublicKey();
Algorithm algorithm = Algorithm.RSA256(publicKey, null);
JWTVerifier verifier = JWT.require(algorithm).build();
DecodedJWT decoded = verifier.verify(token);
```

## 비교표

| 구분 | 대칭키 (HMAC) | 비대칭키 (RSA) |
|------|--------------|---------------|
| **키 개수** | 1개 (Secret Key) | 2개 (Public/Private Key) |
| **성능** | 빠름 | 느림 (약 100-1000배) |
| **리소스 사용** | 낮음 | 높음 |
| **키 관리** | 어려움 (공유 필요) | 쉬움 (공개키 공개 가능) |
| **보안성** | 키 유출 시 위험 | 개인키 보호만 필요 |
| **분산 환경** | 부적합 | 적합 |
| **확장성** | 우수 | 제한적 |
| **구현 복잡도** | 낮음 | 높음 |

## 실제 사용 시나리오

### 시나리오 1: 단일 애플리케이션 (모놀리식)

```
클라이언트 → 애플리케이션 (토큰 발급 및 검증)
```

**대칭키 방식이 적합:**
- 같은 애플리케이션에서 토큰을 발급하고 검증
- 키 관리가 단순함
- 높은 성능이 필요

```java
// 단일 애플리케이션에서 사용
String secretKey = "application-secret-key";
// 발급과 검증 모두 같은 키 사용
```

### 시나리오 2: 마이크로서비스 아키텍처

```
인증 서비스 (토큰 발급) → 여러 마이크로서비스 (토큰 검증)
```

**비대칭키 방식이 적합:**
- 인증 서비스만 개인키 보유
- 다른 서비스들은 공개키만으로 검증
- 키 유출 위험 최소화

```java
// 인증 서비스 (개인키로 발급)
RSAPrivateKey privateKey = loadPrivateKey();
String token = JWT.create().sign(Algorithm.RSA256(null, privateKey));

// 다른 마이크로서비스 (공개키로 검증)
RSAPublicKey publicKey = loadPublicKey();
JWT.require(Algorithm.RSA256(publicKey, null)).build().verify(token);
```

### 시나리오 3: 외부 API 제공

```
내부 서비스 (토큰 발급) → 외부 클라이언트 (토큰 검증)
```

**비대칭키 방식이 적합:**
- 공개키를 외부에 공개해도 안전
- 개인키는 내부에서만 보관
- 키 유출 위험 없음

## 성능 비교

실제 벤치마크 결과 (10,000개 토큰 생성/검증):

```
대칭키 (HMAC-SHA256):
  - 생성: ~50ms
  - 검증: ~50ms
  - 총: ~100ms

비대칭키 (RSA256):
  - 생성: ~5,000ms
  - 검증: ~500ms
  - 총: ~5,500ms
```

**비대칭키는 대칭키보다 약 50-100배 느립니다.**

## 현재 프로젝트에서의 선택

### Yellow Store 프로젝트의 경우

**마이크로서비스 아키텍처**를 채택하고 있으므로, 다음과 같은 고려사항이 있습니다:

1. **인증 서비스 분리**: 별도의 인증 서비스에서 토큰 발급
2. **여러 서비스에서 검증**: User Service, Product Service 등에서 토큰 검증
3. **키 관리**: 각 서비스에 비밀키를 배포하는 것은 보안상 위험

**따라서 비대칭키 방식(RSA)을 권장합니다.**

### 구현 예제

```java
// 인증 서비스 (토큰 발급)
@Service
public class AuthService {
    
    @Value("${jwt.private-key}")
    private String privateKeyPath;
    
    public String generateToken(User user) {
        RSAPrivateKey privateKey = loadPrivateKey(privateKeyPath);
        Algorithm algorithm = Algorithm.RSA256(null, privateKey);
        
        return JWT.create()
            .withSubject(user.getId())
            .withClaim("role", user.getRole())
            .withExpiresAt(new Date(System.currentTimeMillis() + 3600000))
            .sign(algorithm);
    }
}

// 다른 마이크로서비스 (토큰 검증)
@Component
public class JwtValidator {
    
    @Value("${jwt.public-key}")
    private String publicKeyPath;
    
    public DecodedJWT validateToken(String token) {
        RSAPublicKey publicKey = loadPublicKey(publicKeyPath);
        Algorithm algorithm = Algorithm.RSA256(publicKey, null);
        JWTVerifier verifier = JWT.require(algorithm).build();
        
        return verifier.verify(token);
    }
}
```

## 하이브리드 접근법

일부 시스템에서는 두 방식을 함께 사용하기도 합니다:

- **내부 서비스 간 통신**: 대칭키 (성능 우선)
- **외부 API 제공**: 비대칭키 (보안 우선)
- **인증 토큰**: 비대칭키 (분산 환경)
- **리프레시 토큰**: 대칭키 (성능)

## 결론

### 대칭키를 선택해야 하는 경우

✅ 단일 애플리케이션에서 토큰 발급 및 검증
✅ 높은 성능이 최우선
✅ 키 관리가 단순한 환경
✅ 대량의 토큰 처리 필요

### 비대칭키를 선택해야 하는 경우

✅ 마이크로서비스 아키텍처
✅ 토큰 발급과 검증이 다른 서비스에서 이루어짐
✅ 보안성이 최우선
✅ 공개키를 안전하게 공유할 수 있는 환경
✅ 키 회전이 필요한 환경

**Yellow Store 프로젝트에서는 마이크로서비스 아키텍처를 채택하고 있어 비대칭키 방식(RSA256)을 사용하고 있으며, 이를 권장합니다.** 성능보다는 보안성과 분산 환경에서의 키 관리 편의성이 더 중요하기 때문입니다.

---

다음 포스트에서는 **Kafka와 RabbitMQ의 동작 차이**에 대해 다루겠습니다. 두 메시지 브로커의 아키텍처와 사용 시나리오를 비교해보겠습니다. 다음 글에서 만나요! 📨

