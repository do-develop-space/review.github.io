---
layout: post
title: "BFF에서 인증/인가와 보안을 어떻게 설계할까?"
date: 2025-12-05
categories: [architecture]
tags: [BFF, 인증, 인가, 보안, JWT, APIGateway, OAuth2, MSA]
---

# BFF에서 인증/인가와 보안을 어떻게 설계할까?

이전 글에서는 BFF(Backend for Frontend) 패턴의 개념과 API Gateway와의 차이를 정리했습니다.  
이번 글에서는 그 연장선으로, **BFF 레이어에서 인증/인가와 보안을 어디까지 책임져야 하는지**를 정리해보려고 합니다.

단일 백엔드 구조에서는 “백엔드가 인증/인가 다 한다”라고 생각하기 쉽지만,  
API Gateway, Auth 서버, BFF, 도메인 마이크로서비스가 분리된 MSA 환경에서는 **역할 분리가 더 중요해집니다.**

---

## 1. 큰 그림: 누가 무엇을 책임져야 할까?

먼저 전형적인 구조를 하나 떠올려 봅시다.

```
Client (Web/Mobile)
    ↓
API Gateway (인증 토큰 검증, 라우팅, Rate Limit)
    ↓
BFF (프론트 전용 조합 로직, 권한 체크, 데이터 필터링)
    ↓
Domain Services (User, Order, Product 등)
```

역할을 나누면 대략 이렇게 볼 수 있습니다.

- **API Gateway**
  - 토큰 형식/서명 검증(JWT Signature 검증, 만료 여부)
  - 요청 라우팅, Rate Limiting, IP/Geo 기반 간단 차단
  - 공통 로깅·모니터링
- **Auth 서버 (또는 IDP)**
  - 로그인/로그아웃, 토큰 발급/갱신
  - 사용자, 권한(roles, scopes) 관리
- **BFF**
  - 프론트엔드 관점의 **권한 체크(Authorization)**
  - 현재 사용자 컨텍스트 기반 데이터 필터링 (본인 것만 보이기 등)
  - 민감 정보 마스킹, 응답 스키마 최소화
- **도메인 마이크로서비스**
  - 핵심 도메인 규칙 기반의 **2차 권한 체크**
  - 리소스 소유권 확인, 도메인 규칙(예: “취소는 배송 전까지만 가능”) 적용

핵심은 **“인증은 최대한 앞단에서, 인가는 여러 레이어에서 방어적으로”** 라는 원칙으로 볼 수 있습니다.

---

## 2. BFF에서 처리해야 하는 인증/인가 책임

### 1) 인증 정보 해석 (토큰 파싱, 사용자 컨텍스트 생성)

API Gateway에서 JWT 서명·만료 검증을 했더라도,  
BFF에서는 토큰에서 다음 정보를 꺼내 **애플리케이션 컨텍스트로 만드는 작업**이 필요합니다.

- `userId`, `username`, `email`
- `roles` (예: `ROLE_USER`, `ROLE_ADMIN`)
- `scopes` (예: `orders:read`, `orders:write`)

Spring Security 기준으로는 보통 다음과 같은 구조가 됩니다.

```java
@RestController
@RequestMapping("/api/web")
public class WebOrderBffController {

    @GetMapping("/me")
    public MeResponse me(@AuthenticationPrincipal UserPrincipal principal) {
        return MeResponse.from(principal);
    }
}
```

- `UserPrincipal` 안에 토큰에서 파싱한 사용자 정보와 권한 정보를 담아두고,
- BFF 전체에서 이를 활용해 **권한 체크 및 데이터 필터링**을 수행합니다.

### 2) 프론트 단위 권한 체크 (화면·기능 수준)

도메인 서비스는 “이 주문을 취소해도 되는가?” 같은 **도메인 규칙**에 집중하는 것이 좋습니다.  
BFF는 그보다 한 단계 위에서 **“어떤 버튼/페이지에 접근 가능한가?”** 같은 프론트 단위 권한을 다루기 좋습니다.

예를 들어:

- 일반 사용자는 `/admin/**` BFF API에 접근 불가
- `ROLE_MANAGER` 이상만 특정 통계 화면 API 호출 가능

```java
@RestController
@RequestMapping("/api/admin")
@PreAuthorize("hasRole('ADMIN') or hasRole('MANAGER')")
public class AdminDashboardBffController {

    @GetMapping("/dashboard")
    public AdminDashboardResponse getDashboard() {
        // 여러 도메인 서비스에서 데이터 조합
    }
}
```

이렇게 하면 **프론트 전용 API에 대한 권한 정책**을 BFF 레이어에 모을 수 있습니다.

### 3) 데이터 레벨 필터링 (본인 것만 보이게)

같은 BFF API라도, **사용자에 따라 보이는 데이터 범위**가 달라질 수 있습니다.

- 일반 사용자: 본인 주문만 조회
- 관리자: 전체 주문 조회 가능

```java
@GetMapping("/orders")
public List<OrderSummaryResponse> getOrders(@AuthenticationPrincipal UserPrincipal user) {
    if (user.hasRole("ADMIN")) {
        return orderClient.getAllOrders();
    }
    return orderClient.getOrdersByUserId(user.getId());
}
```

이처럼 BFF에서 **역할에 따라 어떤 도메인 API를 호출할지 분기**해 줄 수 있습니다.

---

## 3. JWT, 세션, 쿠키: BFF에서 어떻게 사용할까?

### 1) JWT + Bearer 토큰 방식

모바일 앱·SPA 환경에서 흔히 쓰는 방식입니다.

- 클라이언트가 `Authorization: Bearer <JWT>` 헤더로 요청
- API Gateway에서 1차 검증 후 BFF로 전달
- BFF는 토큰 payload를 해석해 `UserPrincipal` 생성

장점:

- 마이크로서비스 간에 인증 정보를 **헤더로 간편하게 전파** 가능
- 무상태(Stateless) 구조 구현 용이

주의할 점:

- 토큰 탈취 시 피해가 크므로, **만료 시간·재발급 전략·보관 위치(localStorage vs 쿠키 등)** 를 신중히 설계해야 합니다.

### 2) 세션 + 쿠키 방식 (특히 웹 BFF)

웹 브라우저 기반 서비스에서는 여전히 **세션·쿠키 조합**이 편리할 때가 많습니다.

- 브라우저가 자동으로 쿠키를 전송
- BFF가 세션 저장소(Redis 등)를 조회해 사용자 정보를 복원

구조 예시:

1. 로그인 시 Auth 서버가 세션/쿠키 발급
2. BFF가 세션 ID 쿠키를 받아 세션 저장소에서 사용자 정보 조회
3. 필요한 경우 BFF가 도메인 서비스와 통신할 때는 **내부용 JWT 또는 서비스 계정**을 사용

이 방식은 특히 **동일 도메인·동일 오리진 환경의 웹 서비스**에서 유용합니다.

---

## 4. BFF에서 고려해야 할 보안 포인트

### 1) 민감 정보 노출 최소화

BFF는 프론트에 맞게 응답을 조합하는 레이어이기 때문에,  
**도메인 서비스에서 받은 전체 데이터를 그대로 노출하지 않고, 필요한 필드만 골라서 내려주는 역할**이 특히 중요합니다.

예를 들어:

- 내부 서비스 응답: `userId`, `email`, `phone`, `socialSecurityNumber`, `internalNotes` …
- BFF 응답: `userId`, `email` 정도만 포함

```java
public UserProfileResponse toUserProfile(UserInternalDto internal) {
    return UserProfileResponse.builder()
            .id(internal.getId())
            .email(internal.getEmail())
            // 민감 정보는 절대 노출하지 않음
            .build();
}
```

민감 정보는 도메인 서비스에서 가져오더라도,  
**BFF DTO 설계 단계에서부터 “프론트에 정말 필요한가?”를 한 번 더 필터링**하는 습관이 중요합니다.

### 2) 입력 검증 및 방어 코드

- BFF는 외부에서 직접 요청을 받는 레이어이므로, **입력 검증(Validation)** 을 반드시 수행해야 합니다.
- 예:
  - Path/Query 파라미터 범위·형식 검증
  - JSON Body 필드 검증
  - 예상치 못한 값에 대한 방어 코드(Null, 음수, 과도한 길이 등)

Spring 기준으로는 `@Valid`와 Bean Validation을 적극 활용할 수 있습니다.

```java
@PostMapping("/orders")
public OrderCreateResponse createOrder(
        @AuthenticationPrincipal UserPrincipal user,
        @Valid @RequestBody CreateOrderRequest request
) {
    // ...
}
```

### 3) Rate Limiting·재시도·타임아웃 기본값

Rate Limiting은 보통 API Gateway에서 처리하지만,  
BFF도 **비정상적인 폭주 요청에 대한 2차 방어선**이 될 수 있습니다.

- BFF에서 도메인 서비스 호출 시 **타임아웃, 재시도 횟수, Circuit Breaker** 설정
- 특정 사용자/클라이언트에서 비정상 패턴이 감지되면 **추가 차단 로직** 고려

---

## 5. 정리 및 마무리

이 글에서는 BFF 관점에서 인증/인가와 보안을 어떻게 나눠서 설계할지 정리해 보았습니다.

- 인증은 API Gateway·Auth 서버에서 최대한 앞단에 두고,  
  BFF에서는 **토큰 해석과 사용자 컨텍스트 생성**에 집중합니다.
- 인가는 **BFF(화면/기능 단위)** 와 **도메인 서비스(도메인 규칙 단위)** 에서 함께 방어적으로 수행합니다.
- BFF는 프론트에 가까운 만큼, **민감 정보 최소 노출·입력 검증·에러 처리·타임아웃/재시도 정책**을 신경 써야 합니다.

저는 BFF 레이어에서 인증/인가를 어디까지 맡기고, 어떤 부분을 도메인 서비스·API Gateway와 나눠야 할지 공부하면서, 실제 서비스에 적용 가능한 패턴들을 하나씩 정리해 나가고 있습니다.  
이어지는 다음 글에서는 이런 아키텍처적인 결정과 요구사항을 **GitHub 기반 스펙 문서와 ADR로 어떻게 남기고, 코드·PR 흐름과 연결할지** 정리해보겠습니다. 읽어주셔서 감사합니다! 🔐


