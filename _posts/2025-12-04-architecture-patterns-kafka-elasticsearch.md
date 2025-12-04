---
layout: post
title: "BFF(Backend for Frontend) 패턴: API Gateway와 무엇이 다를까?"
date: 2025-12-04
categories: [architecture]
tags: [BFF, BackendForFrontend, APIGateway, 마이크로서비스, 프론트엔드아키텍처, 모바일백엔드]
---

# BFF(Backend for Frontend) 패턴: API Gateway와 무엇이 다를까?

마이크로서비스 아키텍처에서 모바일 앱, 웹 프론트엔드, 관리자 콘솔 등 **클라이언트 종류가 다양해질수록** “백엔드를 어떻게 나눌 것인가?”라는 고민이 커집니다.  
이때 자주 등장하는 개념이 바로 **BFF(Backend for Frontend) 패턴**입니다. 이름 그대로 **특정 프론트엔드를 위한 전용 백엔드**를 따로 두자는 아이디어입니다.

이 글에서는 BFF 패턴의 개념, API Gateway와의 차이, 언제 쓰면 좋은지, 그리고 간단한 설계 예시까지 정리해보겠습니다.

---

## 1. BFF 패턴이란?

### 정의

**BFF(Backend for Frontend)** 패턴은 다음과 같이 정의할 수 있습니다.

> “특정 프론트엔드(모바일 앱, 웹, 관리자 페이지 등)의 요구사항에 최적화된 전용 백엔드를 별도로 둔다.”

즉, 하나의 거대한 백엔드가 **모든 클라이언트의 요구사항을 처리**하는 대신,

- **모바일 앱용 BFF**
- **웹 프론트엔드용 BFF**
- **관리자 페이지용 BFF**

처럼 **클라이언트 단위로 백엔드를 잘게 나누는** 패턴입니다.

### 왜 필요한가?

단일 백엔드(API 서버)만 두면 다음과 같은 문제가 생깁니다.

- 모바일 전용 필드/화면 구성이 점점 API에 섞여 들어감
- 웹/앱에서 필요한 응답 포맷이 미묘하게 달라져서 if/else 로직이 백엔드에 추가
- 특정 클라이언트의 화면 변경이 공용 백엔드 API 변경으로 이어져, **다른 클라이언트에도 영향** 발생

BFF 패턴은 이런 문제를 줄이기 위해 **“클라이언트별 요구사항은 해당 클라이언트 전용 BFF에서 해결하자”**는 접근입니다.

---

## 2. BFF vs API Gateway: 뭐가 다를까?

헷갈리기 쉬운 개념이 **API Gateway**입니다. 둘 다 “프론트엔드 앞단에 있는 백엔드”처럼 보이기 때문입니다.

### 공통점

- 클라이언트 입장에서는 둘 다 **백엔드의 진입점**처럼 보인다.
- 인증, 로깅, 모니터링, 라우팅 등 **공통 기능을 처리**할 수 있다.

### 차이점 (핵심)

**1) 책임 범위**

- **API Gateway**
  - 주로 “인프라 레벨”의 공통 기능 처리
  - 예: 인증/인가, Rate Limiting, 라우팅, Circuit Breaker, 로깅, 모니터링
- **BFF**
  - “도메인+UI 레벨”의 요구사항 처리
  - 예: 여러 마이크로서비스를 조합해서 **화면에 딱 맞는 DTO로 가공**, 프론트엔드가 쓰기 편한 API 설계

**2) 개수**

- **API Gateway**는 보통 **한 개(또는 소수)** 를 운영
- **BFF**는 **클라이언트별로 여러 개**가 생길 수 있음
  - 예: `web-bff`, `mobile-bff`, `admin-bff` 등

**3) 관점**

- API Gateway는 **플랫폼/인프라 팀** 관점에서 보는 요소
- BFF는 **프론트엔드·프로덕트 팀** 관점에서 보는 요소

---

## 3. 언제 BFF 패턴을 쓰면 좋을까?

### 1) 클라이언트 종류가 많을 때

- 웹, 모바일 앱(iOS/Android), 관리자 웹, 파트너용 포털 등 **여러 타입의 클라이언트**가 있을 때
- 각 클라이언트에서 **화면 구조, UX, 요청 패턴이 많이 다르면** BFF 도입 검토 가치가 큼

### 2) 프론트엔드가 자주 바뀌는 서비스

- UI/UX 실험(A/B 테스트), 화면 개편이 잦은 서비스
- 프론트 요구사항이 자주 바뀌는데, 이를 **공용 백엔드에 계속 붙이다 보면 API 설계가 무너지는 경우**

이때 BFF에서 **프론트 변경을 흡수**하게 하고, 도메인 마이크로서비스는 비교적 안정적인 API를 유지하도록 하는 전략이 유용합니다.

### 3) 프론트에서 여러 서비스 조합이 필요한 경우

- 예: “메인 대시보드”에 주문, 배송, 알림, 추천 상품 등 **여러 서비스 데이터를 한 번에 보여줘야 할 때**
- BFF에서 여러 마이크로서비스를 호출해 **한 번의 API로 합쳐서 내려주기** 좋습니다.

---

## 4. BFF와 마이크로서비스, 어떻게 나눌까?

### 기본 구조 예시

```
┌────────────────┐        ┌────────────────┐
│   Web Frontend │  ───▶  │   Web BFF      │
└────────────────┘        └──────┬─────────┘
                                 │
┌────────────────┐        ┌──────┴─────────┐
│ Mobile App     │  ───▶  │  Mobile BFF   │
└────────────────┘        └──────┬─────────┘
                                 │
                          ┌──────┴────────────┐
                          │  User Service     │
                          │  Order Service    │
                          │  Product Service  │
                          └───────────────────┘
```

- `Web BFF`, `Mobile BFF`는 **프론트엔드에 특화된 조합/가공 로직**을 담당
- 실제 비즈니스 규칙, 트랜잭션, 데이터 저장은 **도메인 마이크로서비스**에서 담당

### 예시: 주문 내역 조회 API

- 웹에서는 상세한 주문 정보(상품 썸네일, 리뷰 버튼 등)가 필요
- 모바일에서는 최소한의 정보(주문 상태, 금액, 대표 상품명 등)만 표시

이때:

- `web-bff`의 `/api/orders`는 **리치한 DTO**로 응답
- `mobile-bff`의 `/api/orders`는 **간단한 DTO**로 응답  
→ 둘 다 내부적으로는 같은 `order-service`, `product-service`를 호출하더라도, **응답 스키마와 조합 로직은 BFF마다 다르게 설계**할 수 있습니다.

---

## 5. Spring 기반으로 BFF를 구현한다면

### 1) 경량 컨트롤러 + 외부 서비스 호출

```java
@RestController
@RequestMapping("/api/web")
public class WebOrderBffController {

    private final OrderClient orderClient;
    private final ProductClient productClient;

    @GetMapping("/orders")
    public List<WebOrderResponse> getOrders(@AuthenticationPrincipal UserPrincipal user) {
        List<OrderSummary> orders = orderClient.getOrders(user.getId());

        return orders.stream()
                .map(order -> {
                    ProductSummary product = productClient.getMainProduct(order.getMainProductId());
                    return WebOrderResponse.of(order, product);
                })
                .toList();
    }
}
```

- `OrderClient`, `ProductClient`는 내부 마이크로서비스를 호출하는 클라이언트
- `WebOrderResponse`는 **웹 화면에 최적화된 DTO**

### 2) API Gateway와 같이 쓸 수도 있다

- 클라우드 환경에서 보통:
  - **외부 진입점**: API Gateway (인증, 라우팅, Rate Limit)
  - **뒤쪽 서비스 중 일부**: BFF
- 구조 예시:

```
Client → API Gateway → Web BFF → 여러 도메인 서비스
                      Mobile BFF → 여러 도메인 서비스
```

API Gateway는 공통 Cross-cutting concern을 다루고,  
BFF는 **각 프론트엔드별 화면과 사용자 경험(UX)에 집중**하도록 역할을 나누는 것이 핵심입니다.

---

## 6. BFF 도입 시 주의할 점

### 1) 서비스 숫자 폭발

- 클라이언트가 많아질수록 BFF도 많아질 수 있습니다.
- 팀 규모, 운영 여력을 고려해서 **어디까지 쪼갤지 기준**을 잡는 것이 중요합니다.

예를 들어:

- 웹/모바일을 한 BFF로 묶고, **관리자용 BFF만 따로 분리**하는 식으로 단계적으로 도입할 수 있습니다.

### 2) 비즈니스 로직이 BFF에 쌓이지 않도록

- BFF에는 **프론트엔드 친화적인 조합/가공 로직**까지만 두고,
- **핵심 비즈니스 규칙, 트랜잭션, 도메인 상태 변경**은 도메인 마이크로서비스에서 처리하는 것이 좋습니다.

BFF가 “또 하나의 거대한 모놀리식 백엔드”가 되지 않도록 의식적으로 선을 그어야 합니다.

---

## 7. 정리 및 마무리

이 글에서는 BFF 패턴의 개념, API Gateway와의 차이, 도입 시나리오, 간단한 구현 예시까지 정리해보았습니다.  
특히 **클라이언트별 요구사항이 계속 추가되고 있는 서비스**라면, BFF 패턴이 프론트엔드와 백엔드 사이의 충돌을 줄이는 좋은 완충 지대가 될 수 있습니다.

저는 BFF 패턴을 실제 MSA 환경에 어떻게 녹여낼지 공부하면서, 웹/모바일을 분리한 설계와 Spring 기반 구현 방식을 하나씩 정리해 나가고 있습니다.  
이어지는 다음 글에서는 BFF 레이어에서 **인증/인가와 보안**을 어디까지 책임지고, API Gateway·Auth 서버와 어떻게 역할을 나눌지 더 구체적으로 정리해보겠습니다. 읽어주셔서 감사합니다! 🙌
