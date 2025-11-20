---
layout: post
title: "DDD와 MSA의 연관관계: 왜 DDD가 마이크로서비스로 가는 길인가"
date: 2025-11-20
categories: [아키텍처]
tags: [DDD, MSA, 마이크로서비스, 헥사고날아키텍처]
---

# DDD와 MSA의 연관관계: 왜 DDD가 마이크로서비스로 가는 길인가

최근 Yellow Store 프로젝트를 진행하면서 DDD(Domain-Driven Design)를 적용해보았습니다. DDD를 공부하고 적용하다 보니 자연스럽게 MSA(Microservices Architecture)와의 연관관계에 대해 생각하게 되었습니다. 이번 포스트에서는 DDD와 MSA가 어떻게 연결되는지, 그리고 왜 DDD가 마이크로서비스로의 전환을 위한 좋은 기반이 되는지 살펴보겠습니다.

## DDD와 MSA의 기본 개념

### DDD (Domain-Driven Design)

DDD는 복잡한 도메인을 모델링하고 소프트웨어를 설계하는 방법론입니다. 핵심 개념은 다음과 같습니다:

- **바운디드 컨텍스트(Bounded Context)**: 도메인 모델이 일관되게 적용되는 경계
- **애그리게이트(Aggregate)**: 일관성 경계를 가진 도메인 객체의 묶음
- **유비쿼터스 언어(Ubiquitous Language)**: 도메인 전문가와 개발자가 공유하는 공통 언어

### MSA (Microservices Architecture)

MSA는 애플리케이션을 작은 독립적인 서비스들로 분해하는 아키텍처 스타일입니다:

- **독립적인 배포**: 각 서비스는 독립적으로 배포 가능
- **기술 다양성**: 서비스마다 다른 기술 스택 사용 가능
- **분산 시스템**: 서비스 간 통신은 네트워크를 통해 이루어짐

## DDD가 MSA로 가는 자연스러운 길인 이유

### 1. 바운디드 컨텍스트 = 마이크로서비스 경계

DDD의 핵심인 **바운디드 컨텍스트**는 MSA의 서비스 경계와 매우 잘 맞습니다.

```
[모놀리식 애플리케이션]
├── Identity & Access Context (회원/인증)
├── Product Context (상품)
├── Payment Context (결제)
└── Order Context (주문)
```

위와 같이 바운디드 컨텍스트로 나눈 구조는, 각 컨텍스트를 독립적인 마이크로서비스로 분리하기에 완벽한 경계가 됩니다.

**Yellow Store 프로젝트에서의 예시:**

```java
// Identity & Access Context
com.example.demo.member/
├── domain/
│   └── Member.java
├── application/
│   └── MemberService.java
└── infrastructure/
    └── MemberRepositoryAdapter.java

// Payment Context  
com.example.demo.payment/
├── domain/
│   └── Payment.java
├── application/
│   └── PaymentService.java
└── infrastructure/
    └── PaymentRepositoryAdapter.java
```

이미 컨텍스트별로 패키지가 분리되어 있어, 나중에 마이크로서비스로 분리할 때 경계가 명확합니다.

### 2. 애그리게이트 = 서비스 간 통신 단위

DDD의 **애그리게이트**는 트랜잭션 경계이자 일관성 단위입니다. MSA에서는 각 서비스가 독립적인 데이터베이스를 가지므로, 애그리게이트 단위로 서비스를 설계하면 자연스럽게 분산 트랜잭션 문제를 피할 수 있습니다.

**예시:**
- `Member` 애그리게이트 → Member Service
- `Product` 애그리게이트 → Product Service
- `Payment` 애그리게이트 → Payment Service

각 애그리게이트는 자신의 데이터를 소유하고, 다른 서비스와는 이벤트나 API를 통해 통신합니다.

### 3. 도메인 이벤트 = 서비스 간 통신 메커니즘

DDD의 **도메인 이벤트**는 MSA에서 서비스 간 비동기 통신을 위한 완벽한 메커니즘입니다.

이렇게 이벤트 기반으로 통신하면 서비스 간 결합도를 낮추고 확장성을 높일 수 있습니다.

## DDD 없이 MSA로 가면?

DDD 없이 기술적인 이유만으로 MSA를 도입하면 다음과 같은 문제가 발생할 수 있습니다:

### 1. 애매한 서비스 경계

어디까지가 하나의 서비스인지 명확하지 않아, 서비스 간 강한 결합이 발생합니다.

### 2. 분산 트랜잭션의 복잡성

서비스 경계가 명확하지 않으면 여러 서비스에 걸친 트랜잭션을 처리해야 하는 상황이 자주 발생합니다.

### 3. 데이터 일관성 문제

서비스 간 데이터 동기화가 복잡해지고, 중복 데이터 관리가 어려워집니다.

## DDD를 먼저 적용하는 것이 좋은 이유

### 1. 점진적 전환 가능

모놀리식 애플리케이션에서도 DDD를 적용하면, 나중에 MSA로 전환할 때 경계가 이미 명확하게 정의되어 있습니다.

**Yellow Store 프로젝트의 경우:**
- 현재는 모놀리식이지만, 바운디드 컨텍스트별로 패키지가 분리되어 있음
- 나중에 각 컨텍스트를 독립 서비스로 분리하기 쉬움

### 2. 도메인 이해도 향상

DDD를 적용하면서 도메인을 깊이 이해하게 되고, 이는 올바른 서비스 분할의 기반이 됩니다.

### 3. 기술 부채 감소

명확한 경계와 책임 분리로 인해 코드의 유지보수성이 향상됩니다.

## 실제 적용 시 고려사항

### 1. 컨텍스트 매핑 전략

서로 다른 바운디드 컨텍스트 간의 관계를 정의해야 합니다:

- **Shared Kernel**: 공유 커널 (공통 코드)
- **Customer-Supplier**: 고객-공급자 관계
- **Conformist**: 순응자 관계
- **Anti-Corruption Layer**: 부패 방지 계층

### 2. 이벤트 기반 통신 설계

서비스 간 통신을 위해 도메인 이벤트를 설계하고, 이벤트 버스를 구축해야 합니다.

### 3. 데이터 일관성 전략

최종 일관성(Eventual Consistency)을 받아들이고, 보상 트랜잭션(Compensating Transaction) 패턴을 고려해야 합니다.

## 결론

DDD와 MSA는 서로 다른 목적을 가진 개념이지만, 매우 잘 맞는 조합입니다:

1. **바운디드 컨텍스트**는 마이크로서비스의 경계가 됩니다
2. **애그리게이트**는 서비스 간 통신의 단위가 됩니다
3. **도메인 이벤트**는 서비스 간 통신 메커니즘이 됩니다

따라서 MSA를 계획하고 있다면, 먼저 모놀리식 애플리케이션에 DDD를 적용해보는 것을 강력히 추천합니다. 이렇게 하면 나중에 MSA로 전환할 때 명확한 경계와 설계 원칙을 가지고 시작할 수 있습니다.

Yellow Store 프로젝트에서도 이 원칙을 따라 DDD를 먼저 적용했고, 향후 필요시 각 바운디드 컨텍스트를 독립적인 마이크로서비스로 분리할 수 있는 기반을 마련했습니다.

---

**참고 자료:**
- Eric Evans, "Domain-Driven Design"
- Sam Newman, "Building Microservices"
- [Yellow Store 프로젝트](https://github.com/do-develop-space/yellow-store)

---

다음 글에서는 **AOP(Aspect-Oriented Programming)**에 대해 다뤄보겠습니다. 관심 있으신 분들은 다음 포스트를 기대해주세요!

감사합니다. 🙏
