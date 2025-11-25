---
layout: post
title: "API Gateway vs Nginx: 마이크로서비스 아키텍처에서의 역할 차이"
date: 2025-11-25
categories: [architecture]
tags: [API Gateway, Nginx, 마이크로서비스, Spring Cloud Gateway, 리버스프록시]
---

# API Gateway vs Nginx: 마이크로서비스 아키텍처에서의 역할 차이

마이크로서비스 아키텍처를 구축할 때 API Gateway와 Nginx는 모두 중요한 역할을 합니다. 하지만 두 기술은 서로 다른 목적과 사용 시점을 가지고 있습니다. 이번 포스트에서는 API Gateway와 Nginx의 차이점을 명확히 하고, 각각의 역할과 언제 사용해야 하는지 알아보겠습니다.

## Nginx란?

Nginx는 고성능 웹 서버이자 리버스 프록시 서버입니다. 주로 다음과 같은 용도로 사용됩니다:

- **정적 파일 서빙**: HTML, CSS, JavaScript, 이미지 등
- **리버스 프록시**: 클라이언트 요청을 백엔드 서버로 전달
- **로드 밸런싱**: 여러 서버에 요청 분산
- **SSL/TLS 종료**: HTTPS 연결 처리
- **캐싱**: 정적 콘텐츠 캐싱

### Nginx의 특징

```nginx
# Nginx 설정 예제
upstream backend {
    server localhost:8080;
    server localhost:8081;
}

server {
    listen 80;
    server_name example.com;

    location / {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

**Nginx의 장점:**
- 매우 빠른 성능 (비동기 이벤트 기반)
- 낮은 메모리 사용량
- 정적 파일 서빙에 최적화
- 간단한 설정

## API Gateway란?

API Gateway는 마이크로서비스 아키텍처에서 모든 클라이언트 요청의 단일 진입점(Single Entry Point) 역할을 하는 서비스입니다. 주로 다음과 같은 기능을 제공합니다:

- **라우팅**: 요청을 적절한 마이크로서비스로 전달
- **인증/인가**: JWT 토큰 검증, API 키 관리
- **로드 밸런싱**: 서비스 인스턴스 간 요청 분산
- **레이트 리미팅**: API 호출 제한
- **서비스 디스커버리**: Eureka, Consul 등과 연동
- **응답 변환**: 요청/응답 데이터 변환
- **모니터링 및 로깅**: API 호출 추적

### Spring Cloud Gateway 예제

```java
@Configuration
public class GatewayConfig {
    
    @Bean
    public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
        return builder.routes()
            .route("user-service", r -> r
                .path("/api/users/**")
                .uri("lb://user-service"))
            .route("product-service", r -> r
                .path("/api/products/**")
                .filters(f -> f
                    .addRequestHeader("X-Request-Id", UUID.randomUUID().toString())
                    .circuitBreaker(config -> config
                        .setName("productCircuitBreaker")))
                .uri("lb://product-service"))
            .build();
    }
}
```

## API Gateway vs Nginx: 핵심 차이점

### 1. 목적과 역할

**Nginx:**
- **웹 서버 및 리버스 프록시**에 초점
- 정적 파일 서빙과 HTTP 요청 라우팅
- 네트워크 레벨의 프록시 기능

**API Gateway:**
- **마이크로서비스 아키텍처**에 특화
- 비즈니스 로직 레벨의 라우팅 및 처리
- 서비스 디스커버리, 인증, 모니터링 등 통합 기능

### 2. 처리 레벨

**Nginx:**
```
클라이언트 → Nginx (리버스 프록시) → 백엔드 서버
           (네트워크 레벨)
```

**API Gateway:**
```
클라이언트 → API Gateway (라우팅, 인증, 변환) → 마이크로서비스
           (애플리케이션 레벨)
```

### 3. 기능 범위

| 기능 | Nginx | API Gateway |
|------|-------|-------------|
| 정적 파일 서빙 | ✅ 최적화됨 | ❌ |
| 리버스 프록시 | ✅ | ✅ |
| 로드 밸런싱 | ✅ | ✅ |
| 서비스 디스커버리 | ❌ | ✅ |
| 인증/인가 | 제한적 | ✅ 통합 |
| 레이트 리미팅 | 제한적 | ✅ |
| 서킷 브레이커 | ❌ | ✅ |
| 요청/응답 변환 | 제한적 | ✅ |

### 4. 설정 복잡도

**Nginx:**
- 설정 파일 기반 (nginx.conf)
- 정적 설정
- 재시작 필요

**API Gateway (Spring Cloud Gateway):**
- 코드 기반 또는 설정 파일
- 동적 라우팅 가능
- 핫 리로드 가능

## 실제 사용 시나리오

### 시나리오 1: 단순 웹 애플리케이션

```
클라이언트 → Nginx → Spring Boot 애플리케이션
```

**Nginx 사용이 적합한 경우:**
- 단일 애플리케이션 또는 모놀리식 아키텍처
- 정적 파일 서빙이 필요한 경우
- 간단한 리버스 프록시만 필요한 경우

### 시나리오 2: 마이크로서비스 아키텍처

```
클라이언트 → Nginx → API Gateway → 마이크로서비스들
           (SSL 종료)  (라우팅, 인증)
```

**API Gateway 사용이 적합한 경우:**
- 여러 마이크로서비스가 있는 경우
- 서비스 디스커버리가 필요한 경우
- 통합 인증/인가가 필요한 경우
- API 버전 관리가 필요한 경우

## 함께 사용하기: 하이브리드 아키텍처

실제 프로덕션 환경에서는 두 기술을 **함께 사용**하는 것이 일반적입니다:

```
인터넷
  ↓
Nginx (SSL 종료, 정적 파일 서빙)
  ↓
API Gateway (라우팅, 인증, 서비스 디스커버리)
  ↓
마이크로서비스들
```

**역할 분담:**
- **Nginx**: 
  - SSL/TLS 종료
  - 정적 파일 서빙 (이미지, CSS, JS)
  - 기본적인 로드 밸런싱
  - DDoS 방어

- **API Gateway**:
  - 마이크로서비스 라우팅
  - JWT 토큰 검증
  - 서비스 디스커버리 연동
  - API 레이트 리미팅
  - 서킷 브레이커

### 실제 구성 예제

```nginx
# Nginx 설정
server {
    listen 443 ssl;
    server_name api.example.com;

    # SSL 설정
    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;

    # 정적 파일
    location /static/ {
        root /var/www/static;
    }

    # API Gateway로 프록시
    location /api/ {
        proxy_pass http://api-gateway:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

```yaml
# Spring Cloud Gateway 설정
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
                redis-rate-limiter.replenishRate: 10
                redis-rate-limiter.burstCapacity: 20
```

## 언제 무엇을 선택해야 할까?

### Nginx를 선택해야 하는 경우

✅ **단일 애플리케이션** 또는 모놀리식 아키텍처
✅ **정적 파일 서빙**이 주요 요구사항
✅ **간단한 리버스 프록시**만 필요
✅ **높은 성능**이 최우선 (정적 콘텐츠)
✅ **비용 효율적**인 솔루션 필요

### API Gateway를 선택해야 하는 경우

✅ **마이크로서비스 아키텍처** 구축
✅ **여러 서비스**를 통합 관리 필요
✅ **서비스 디스커버리** 연동 필요
✅ **통합 인증/인가** 필요
✅ **API 버전 관리** 및 **레이트 리미팅** 필요
✅ **서킷 브레이커** 등 고급 기능 필요

### 함께 사용해야 하는 경우

✅ **프로덕션 환경**에서 안정성과 성능 모두 필요
✅ **SSL 종료**와 **마이크로서비스 라우팅** 모두 필요
✅ **정적 파일 서빙**과 **동적 API 라우팅** 모두 필요

## 결론

Nginx와 API Gateway는 서로 다른 목적을 가진 도구입니다:

| 구분 | Nginx | API Gateway |
|------|-------|-------------|
| **주요 역할** | 웹 서버, 리버스 프록시 | 마이크로서비스 게이트웨이 |
| **처리 레벨** | 네트워크 레벨 | 애플리케이션 레벨 |
| **정적 파일** | ✅ 최적화 | ❌ |
| **서비스 디스커버리** | ❌ | ✅ |
| **인증/인가** | 제한적 | ✅ 통합 |
| **복잡도** | 낮음 | 높음 |

**권장 사항:**
- **단순 애플리케이션** → Nginx만 사용
- **마이크로서비스** → API Gateway 사용
- **프로덕션 환경** → Nginx + API Gateway 함께 사용

Yellow Store 프로젝트에서는 마이크로서비스 아키텍처를 채택하고 있어 Spring Cloud Gateway를 API Gateway로 사용하고 있으며, 프로덕션 환경에서는 Nginx를 앞단에 배치하여 SSL 종료와 정적 파일 서빙을 담당하도록 구성할 예정입니다.

---

다음 포스트에서는 **JWT의 비대칭키와 대칭키 방식**에 대해 다루겠습니다. 각 방식의 차이점과 현재 프로젝트에서 어떤 방식을 사용해야 하는지 알아보겠습니다. 많은 관심 부탁드립니다! 🔐

