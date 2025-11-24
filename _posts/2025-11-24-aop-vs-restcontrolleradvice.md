---
layout: post
title: "AOP vs RestControllerAdvice: 언제 어떤 방식을 사용해야 할까?"
date: 2025-11-24
categories: [programming]
tags: [AOP, RestControllerAdvice, 예외처리, Spring, 횡단관심사]
---

# AOP vs RestControllerAdvice: 언제 어떤 방식을 사용해야 할까?

이전 포스트에서 AOP(Aspect-Oriented Programming)에 대해 알아보았습니다. Spring Boot에서 예외 처리나 공통 로직을 구현할 때 AOP 외에도 `@RestControllerAdvice`를 사용하는 방법이 있습니다. 

두 방식 모두 횡단 관심사를 처리하는 방법이지만, 각각의 특징과 사용 시점이 다릅니다. 이번 포스트에서는 AOP와 `@RestControllerAdvice`의 차이점을 명확히 하고, 프로젝트에서 어떤 방식을 선택해야 하는지 알아보겠습니다.

## RestControllerAdvice란?

`@RestControllerAdvice`는 Spring에서 제공하는 전역 예외 처리 및 응답 공통화를 위한 어노테이션입니다. 주로 다음과 같은 용도로 사용됩니다:

- **전역 예외 처리**: 모든 컨트롤러에서 발생하는 예외를 한 곳에서 처리
- **공통 응답 포맷**: 모든 API 응답을 일관된 형식으로 변환
- **요청/응답 로깅**: 모든 요청과 응답에 대한 공통 로깅

### 기본 사용 예제

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(IllegalArgumentException.class)
    public ResponseEntity<ErrorResponse> handleIllegalArgument(
            IllegalArgumentException e) {
        ErrorResponse error = new ErrorResponse(
            "INVALID_INPUT", 
            e.getMessage()
        );
        return ResponseEntity
            .status(HttpStatus.BAD_REQUEST)
            .body(error);
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleException(Exception e) {
        ErrorResponse error = new ErrorResponse(
            "INTERNAL_ERROR", 
            "서버 내부 오류가 발생했습니다."
        );
        return ResponseEntity
            .status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(error);
    }
}
```

## AOP와 RestControllerAdvice의 차이점

### 1. 적용 범위

**AOP:**
- 메서드 실행 전/후, 예외 발생 시 등 **다양한 시점**에 로직 삽입 가능
- 컨트롤러뿐만 아니라 **서비스, 리포지토리 등 모든 레이어**에 적용 가능
- Pointcut 표현식으로 **세밀한 타겟 지정** 가능

**RestControllerAdvice:**
- **컨트롤러 레이어에서 발생하는 예외**만 처리
- 메서드 실행 전/후 로직 삽입 불가 (예외 발생 시에만 동작)
- 모든 컨트롤러에 자동 적용

### 2. 실행 시점

**AOP:**
```
요청 → AOP Before → 컨트롤러 메서드 실행 → AOP After → 응답
                    ↓ (예외 발생 시)
                    AOP AfterThrowing
```

**RestControllerAdvice:**
```
요청 → 컨트롤러 메서드 실행 → (예외 발생 시) → @ExceptionHandler 실행 → 응답
```

### 3. 사용 목적

**AOP:**
- 로깅, 트랜잭션 관리, 성능 측정, 보안 검증 등 **다양한 횡단 관심사** 처리
- 메서드 실행 전후의 **공통 로직** 처리
- 비즈니스 로직과 완전히 분리된 **기술적 관심사** 처리

**RestControllerAdvice:**
- **예외 처리**에 특화
- **HTTP 응답 포맷 통일**
- 컨트롤러 레이어의 **에러 핸들링** 전담

## 실제 사용 사례 비교

### 예외 처리: RestControllerAdvice가 적합한 경우

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(EntityNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(
            EntityNotFoundException e) {
        return ResponseEntity
            .status(HttpStatus.NOT_FOUND)
            .body(new ErrorResponse("NOT_FOUND", e.getMessage()));
    }
    
    @ExceptionHandler(ValidationException.class)
    public ResponseEntity<ErrorResponse> handleValidation(
            ValidationException e) {
        return ResponseEntity
            .status(HttpStatus.BAD_REQUEST)
            .body(new ErrorResponse("VALIDATION_ERROR", e.getMessage()));
    }
}
```

**왜 RestControllerAdvice가 적합한가?**
- 예외 처리에만 집중할 수 있음
- HTTP 응답 구조를 명확하게 정의 가능
- 컨트롤러 레이어의 예외만 처리하면 되므로 단순함

### 로깅: AOP가 적합한 경우

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
            
            log.info("Method: {} executed in {} ms", 
                joinPoint.getSignature(), executionTime);
            
            return result;
        } catch (Exception e) {
            log.error("Error in method: {}", joinPoint.getSignature(), e);
            throw e;
        }
    }
}
```

**왜 AOP가 적합한가?**
- 서비스 레이어에도 적용 가능
- 메서드 실행 전후 모두 로깅 가능
- 예외 발생 시에도 로깅 가능

### 트랜잭션 관리: AOP가 적합한 경우

```java
@Aspect
@Component
public class TransactionAspect {
    
    @Around("@annotation(com.example.demo.annotation.Transactional)")
    public Object manageTransaction(ProceedingJoinPoint joinPoint) throws Throwable {
        // 트랜잭션 시작
        TransactionStatus status = transactionManager.getTransaction(
            new DefaultTransactionDefinition()
        );
        
        try {
            Object result = joinPoint.proceed();
            transactionManager.commit(status);
            return result;
        } catch (Exception e) {
            transactionManager.rollback(status);
            throw e;
        }
    }
}
```

**왜 AOP가 적합한가?**
- 서비스 레이어에 적용해야 함
- 메서드 실행 전후 트랜잭션 관리 필요
- Spring의 `@Transactional`도 내부적으로 AOP로 구현됨

## 언제 어떤 방식을 선택해야 할까?

### RestControllerAdvice를 선택해야 하는 경우

✅ **예외 처리에만 집중**하고 싶을 때
- HTTP 응답 포맷을 통일하고 싶을 때
- 컨트롤러 레이어의 예외만 처리하면 될 때
- 코드가 단순하고 이해하기 쉬워야 할 때

**예시:**
- API 응답을 `{ "code": "ERROR_CODE", "message": "..." }` 형식으로 통일
- 비즈니스 예외를 HTTP 상태 코드로 매핑
- 전역 예외 로깅 및 모니터링

### AOP를 선택해야 하는 경우

✅ **다양한 레이어**에 공통 로직을 적용해야 할 때
- 메서드 실행 **전후** 모두 처리해야 할 때
- 예외 처리 외의 **다른 횡단 관심사**를 처리해야 할 때
- **세밀한 타겟 지정**이 필요할 때

**예시:**
- 서비스 레이어의 메서드 실행 시간 측정
- 특정 어노테이션이 있는 메서드만 로깅
- 트랜잭션 관리, 캐싱, 보안 검증 등

## 두 방식을 함께 사용하기

실제 프로젝트에서는 두 방식을 **상호 보완적으로** 사용하는 것이 일반적입니다:

```java
// AOP: 서비스 레이어 로깅 및 성능 측정
@Aspect
@Component
public class ServiceLoggingAspect {
    @Around("execution(* com.example.demo.service.*.*(..))")
    public Object logService(ProceedingJoinPoint joinPoint) throws Throwable {
        // 로깅 로직
    }
}

// RestControllerAdvice: 컨트롤러 레이어 예외 처리
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ErrorResponse> handleBusinessException(
            BusinessException e) {
        // 예외 처리 로직
    }
}
```

이렇게 하면:
- **AOP**: 비즈니스 로직 레이어의 공통 관심사 처리
- **RestControllerAdvice**: HTTP 레이어의 예외 처리 및 응답 포맷 통일

## 결론

AOP와 `@RestControllerAdvice`는 각각 다른 목적과 사용 시점을 가지고 있습니다:

| 구분 | AOP | RestControllerAdvice |
|------|-----|---------------------|
| **적용 범위** | 모든 레이어 | 컨트롤러 레이어 |
| **실행 시점** | 메서드 전/후/예외 | 예외 발생 시 |
| **주요 용도** | 로깅, 트랜잭션, 보안 등 | 예외 처리, 응답 포맷 통일 |
| **복잡도** | 높음 (Pointcut 표현식) | 낮음 (단순 예외 처리) |

**권장 사항:**
- **예외 처리 및 HTTP 응답 통일** → `@RestControllerAdvice`
- **로깅, 트랜잭션, 성능 측정 등** → AOP
- **대부분의 프로젝트** → 두 방식 모두 사용

Yellow Store 프로젝트에서는 서비스 레이어의 로깅과 성능 측정은 AOP로, 컨트롤러 레이어의 예외 처리는 `@RestControllerAdvice`로 구현하여 각각의 장점을 활용하고 있습니다.

---

다음 포스트에서는 **API Gateway 모듈과 nginx의 역할 차이**에 대해 다루겠습니다. 마이크로서비스 아키텍처에서 두 방식이 어떻게 다른지, 언제 어떤 방식을 선택해야 하는지 알아보겠습니다. 많은 관심 부탁드립니다! 🚀

