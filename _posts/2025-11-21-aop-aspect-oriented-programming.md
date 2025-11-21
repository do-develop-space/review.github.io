---
layout: post
title: "AOP(Aspect-Oriented Programming): 횡단 관심사의 우아한 해결책"
date: 2025-11-21
categories: [programming]
tags: [AOP, Spring AOP, 횡단관심사, 관점지향프로그래밍]
---

# AOP(Aspect-Oriented Programming): 횡단 관심사의 우아한 해결책

Spring Boot 프로젝트를 개발하다 보면 로깅, 트랜잭션 관리, 예외 처리, 보안 검증 등 여러 컨트롤러나 서비스에 공통으로 적용해야 하는 기능들이 있습니다. 이런 기능들을 각 메서드마다 반복해서 작성하다 보면 코드 중복이 발생하고, 유지보수가 어려워집니다. 

이런 문제를 해결하기 위한 방법이 바로 **AOP(Aspect-Oriented Programming, 관점 지향 프로그래밍)**입니다. 이번 포스트에서는 AOP의 개념과 Spring AOP의 활용 방법에 대해 알아보겠습니다.

## AOP란?

AOP는 **횡단 관심사(Cross-Cutting Concerns)**를 모듈화하여 코드의 중복을 줄이고 관심사를 분리하는 프로그래밍 패러다임입니다.

### 횡단 관심사(Cross-Cutting Concerns)란?

**횡단 관심사**는 여러 클래스나 모듈을 "가로지르며(횡단하며)" 반복적으로 나타나는 공통 기능을 말합니다.

#### 왜 "횡단"이라고 부를까?

일반적인 비즈니스 로직은 다음과 같이 **수직적으로** 구성됩니다:

```
MemberService (회원 관리)
    ├── create()
    ├── update()
    └── delete()

ProductService (상품 관리)
    ├── create()
    ├── update()
    └── delete()

PaymentService (결제 관리)
    ├── process()
    └── refund()
```

하지만 로깅, 트랜잭션, 예외 처리 같은 기능들은 이 모든 서비스를 **가로지르며(횡단하며)** 적용됩니다:

```
[로깅] ────────────────→ 모든 서비스에 적용
[트랜잭션] ─────────────→ 모든 서비스에 적용
[예외 처리] ────────────→ 모든 서비스에 적용
```

이런 특징 때문에 "횡단 관심사"라고 부릅니다.

#### 횡단 관심사의 예시:

- **로깅**: 모든 메서드의 실행 시간, 파라미터, 결과를 기록
- **트랜잭션 관리**: 데이터베이스 작업의 일관성 보장
- **예외 처리**: 공통적인 예외 처리 로직
- **보안 검증**: 인증/인가 체크
- **성능 모니터링**: 메서드 실행 시간 측정
- **캐싱**: 반복적인 데이터 조회 최적화

이런 기능들을 각 메서드에 직접 작성하면 코드가 복잡해지고, 비즈니스 로직과 섞여서 가독성이 떨어집니다.

## AOP의 핵심 개념

### 1. Aspect (관점)

횡단 관심사를 모듈화한 단위입니다. 예를 들어, "로깅 관점", "트랜잭션 관점" 등이 있습니다.

### 2. Join Point (조인 포인트)

Aspect를 적용할 수 있는 지점입니다. Spring AOP에서는 **메서드 실행 시점**이 조인 포인트입니다.

### 3. Pointcut (포인트컷)

조인 포인트 중에서 실제로 Aspect를 적용할 대상을 선택하는 표현식입니다.

```java
@Pointcut("execution(* com.example.demo.service.*.*(..))")
public void serviceMethods() {}
```

위 예시는 `com.example.demo.service` 패키지의 모든 메서드를 대상으로 합니다.

### 4. Advice (어드바이스)

실제로 실행될 로직입니다. Spring AOP에서는 다음과 같은 어드바이스 타입을 제공합니다:

- **@Before**: 메서드 실행 전
- **@After**: 메서드 실행 후 (성공/실패 무관)
- **@AfterReturning**: 메서드가 정상적으로 반환된 후
- **@AfterThrowing**: 메서드에서 예외가 발생한 후
- **@Around**: 메서드 실행 전후 모두 제어 (가장 강력함)

### 5. Weaving (위빙)

Aspect를 실제 코드에 적용하는 과정입니다. Spring AOP는 **프록시 기반**으로 동작하므로 런타임에 위빙이 이루어집니다.

## Spring AOP 동작 원리

Spring AOP는 **프록시 패턴**을 사용합니다:

```
[Client] 
    ↓ 호출
[프록시 객체] ← AOP 적용
    ↓ 위임
[실제 대상 객체]
```

1. Spring이 대상 객체를 감싸는 프록시 객체를 생성합니다.
2. 클라이언트는 프록시 객체를 통해 메서드를 호출합니다.
3. 프록시가 어드바이스를 실행한 후 실제 객체의 메서드를 호출합니다.

## 실제 사용 예시

### 예시 1: 로깅 AOP

```java
@Aspect
@Component
public class LoggingAspect {
    
    private static final Logger log = LoggerFactory.getLogger(LoggingAspect.class);
    
    @Around("execution(* com.example.demo.service.*.*(..))")
    public Object logExecutionTime(ProceedingJoinPoint joinPoint) throws Throwable {
        long start = System.currentTimeMillis();
        
        try {
            Object result = joinPoint.proceed();
            long executionTime = System.currentTimeMillis() - start;
            
            log.info("{} executed in {} ms", 
                joinPoint.getSignature(), executionTime);
            
            return result;
        } catch (Exception e) {
            log.error("Error in {}", joinPoint.getSignature(), e);
            throw e;
        }
    }
}
```

### 예시 2: 트랜잭션 관리

Spring의 `@Transactional` 어노테이션도 내부적으로 AOP를 사용합니다:

```java
@Service
public class MemberService {
    
    @Transactional
    public MemberResponse create(MemberRequest request) {
        // 트랜잭션이 자동으로 시작됨
        Member member = memberRepository.save(convertToMember(request));
        // 예외 발생 시 자동 롤백
        return convertToResponse(member);
    }
}
```

### 예시 3: 예외 처리 AOP

```java
@Aspect
@Component
public class ExceptionHandlingAspect {
    
    @AfterThrowing(
        pointcut = "execution(* com.example.demo.service.*.*(..))",
        throwing = "ex"
    )
    public void handleException(JoinPoint joinPoint, Exception ex) {
        log.error("Exception in {}: {}", 
            joinPoint.getSignature(), ex.getMessage());
        
        // 예외를 공통 응답 형식으로 변환
        // 또는 알림 발송 등
    }
}
```

## AOP의 장점

### 1. 관심사의 분리 (Separation of Concerns)

비즈니스 로직과 횡단 관심사를 분리하여 코드의 가독성을 높입니다.

**AOP 없이:**
```java
public MemberResponse create(MemberRequest request) {
    log.info("Creating member: {}", request);  // 로깅
    long start = System.currentTimeMillis();    // 성능 측정
    
    try {
        Member member = memberRepository.save(...);  // 비즈니스 로직
        log.info("Member created in {} ms", ...);    // 로깅
        return convertToResponse(member);
    } catch (Exception e) {
        log.error("Error creating member", e);       // 예외 처리
        throw e;
    }
}
```

**AOP 사용:**
```java
public MemberResponse create(MemberRequest request) {
    Member member = memberRepository.save(...);  // 비즈니스 로직만
    return convertToResponse(member);
}
```

### 2. 코드 재사용성

한 번 작성한 Aspect를 여러 곳에 적용할 수 있습니다.

### 3. 유지보수성 향상

횡단 관심사 로직을 한 곳에서 관리하므로 변경이 필요할 때 수정이 용이합니다.

### 4. 비즈니스 로직의 단순화

핵심 비즈니스 로직에만 집중할 수 있어 코드가 깔끔해집니다.

## AOP 사용 시 주의사항

### 1. 프록시 제한사항

Spring AOP는 프록시 기반이므로, 같은 클래스 내부에서 메서드를 호출할 때는 AOP가 적용되지 않습니다:

```java
@Service
public class MemberService {
    
    public void methodA() {
        this.methodB();  // AOP 적용 안 됨!
    }
    
    @Transactional
    public void methodB() {
        // ...
    }
}
```

### 2. 성능 오버헤드

프록시 객체를 생성하고 어드바이스를 실행하는 과정에서 약간의 성능 오버헤드가 발생합니다. 하지만 대부분의 경우 무시할 수 있는 수준입니다.

### 3. 디버깅의 복잡성

프록시를 통해서 실행되므로 스택 트레이스가 복잡해질 수 있습니다.

## Yellow Store 프로젝트에서의 활용

Yellow Store 프로젝트에서는 다음과 같은 AOP 활용이 가능합니다:

1. **로깅 AOP**: 모든 서비스 메서드의 실행 시간과 파라미터 로깅
2. **예외 처리 AOP**: 공통 예외 처리 및 응답 변환
3. **트랜잭션 관리**: `@Transactional`을 통한 자동 트랜잭션 관리
4. **인증/인가 AOP**: 특정 메서드에 대한 권한 검증

예를 들어, 현재 프로젝트에 있는 `NotFoundException`과 `UnAuthorizationException` 같은 예외들을 AOP로 처리하면 더욱 깔끔한 코드를 작성할 수 있습니다.

## 결론

AOP는 횡단 관심사를 효과적으로 처리할 수 있는 강력한 도구입니다. Spring AOP를 활용하면:

- 코드 중복을 줄이고
- 비즈니스 로직에 집중하며
- 유지보수성을 높일 수 있습니다

다만, AOP를 남용하면 오히려 코드를 이해하기 어렵게 만들 수 있으므로, 적절한 상황에서만 사용하는 것이 중요합니다.

---

다음 글에서는 **AOP와 @RestControllerAdvice의 비교**에 대해 다뤄보겠습니다. 두 방식의 차이점과 각각의 사용 시점을 알아보고, 어떤 상황에서 어떤 방식을 선택해야 하는지 살펴보겠습니다. 관심 있으신 분들은 다음 포스트를 기대해주세요!

감사합니다. 🙏

