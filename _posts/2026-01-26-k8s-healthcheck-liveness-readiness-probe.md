---
layout: post
title: "Kubernetes Health Check 완전 정리: Liveness / Readiness / Startup Probe"
date: 2026-01-26
categories: [kubernetes, reliability]
tags: [Kubernetes, HealthCheck, LivenessProbe, ReadinessProbe, StartupProbe, 고가용성, 장애복구]
---

이전 글에서 **Pod Affinity / Pod AntiAffinity**를 이용해 다중 서버 환경에서 Pod를 어떻게 배치할지 정리했습니다.  
이번 글에서는 **Kubernetes Health Check**를 공부하는 느낌으로 정리해보겠습니다.

특히 많이 헷갈리는:

- **Liveness Probe**
- **Readiness Probe**
- **Startup Probe**

세 가지를 **“컨테이너 생명주기 관점”**에서 비교하면서, 실제 설정 예제도 함께 정리하겠습니다.

---

## 1. 왜 Health Check가 중요한가?

Kubernetes는 “컨테이너가 떠 있다 = 정상”이라고 가정하지 않습니다.

실제 운영 환경에서는:

- 프로세스는 살아 있지만
  - 스레드 데드락
  - DB 연결 모두 끊김
  - 내부 에러로 요청을 처리하지 못하는 상태
- 혹은 프로세스가 죽었는데도
  - Kubernetes가 아직 모르는 경우

같은 상황이 생각보다 자주 발생합니다.

**Health Check(Probe)**는 이런 상황을 자동으로 감지해서:

- 컨테이너를 재시작(Liveness)
- Pod를 Endpoints에서 빼서 트래픽을 차단(Readiness)
- 초기에 뜨는 데 오래 걸리는 서비스의 “워밍업 시간”을 보장(Startup)

하도록 도와주는 메커니즘입니다.

---

## 2. Probe 세 가지를 한 번에 비교해 보기

### 2.1 한 줄 정의

- **Liveness Probe**
  - *“이 컨테이너는 여전히 살아있는가?”*
  - 실패하면 **컨테이너 재시작**

- **Readiness Probe**
  - *“이 컨테이너는 지금 트래픽을 받을 준비가 되었는가?”*
  - 실패하면 **Service Endpoints에서 제외** (트래픽 차단)

- **Startup Probe**
  - *“이 컨테이너는 아직 부팅 중인가?”*
  - 초기 부팅이 끝날 때까지 Liveness/Readiness 체크를 **보류**

### 2.2 생명주기 관점에서 보기

컨테이너 관점에서 시간 흐름을 그려보면:

```text
시작 ──▶ (Startup Probe) ──▶ (Liveness + Readiness Probe) ──▶ 종료
```

1. **Startup Probe**
   - 애플리케이션이 초기화되는 동안만 사용
   - 성공할 때까지 Liveness/Readiness는 무시
2. **Liveness Probe**
   - 컨테이너가 “죽었는지”를 꾸준히 체크
   - 실패하면 컨테이너 재시작
3. **Readiness Probe**
   - 컨테이너가 “요청을 받을 준비가 됐는지”를 꾸준히 체크
   - 실패하면 해당 Pod로는 트래픽이 가지 않게 만듦

---

## 3. Probe에서 사용할 수 있는 종류

세 가지 Probe 모두 다음 방식들을 공통으로 사용합니다.

### 3.1 HTTP GET Probe

```yaml
livenessProbe:
  httpGet:
    path: /actuator/health
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  timeoutSeconds: 3
  failureThreshold: 3
```

- 지정한 **HTTP 엔드포인트**를 주기적으로 호출
- 200~399 응답이면 성공, 나머지는 실패로 간주

### 3.2 TCP Socket Probe

```yaml
readinessProbe:
  tcpSocket:
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
```

- 지정한 포트에 **TCP 연결 시도**
- 연결 성공이면 성공, 실패하면 실패

### 3.3 Exec Probe

```yaml
livenessProbe:
  exec:
    command:
      - sh
      - -c
      - "curl -sf http://localhost:8080/actuator/health || exit 1"
  periodSeconds: 10
```

- 컨테이너 내부에서 특정 **명령어를 실행**
- 종료 코드 0이면 성공, 그 외는 실패

HTTP가 가장 많이 쓰이고, TCP는 gRPC나 단순 포트 체크 용도,  
Exec는 아주 특수한 상황(내부 상태 확인 스크립트 등)에 사용합니다.

---

## 4. Liveness Probe: “죽었으면 다시 살려라”

### 4.1 개념

**Liveness Probe**는 컨테이너가 “살아있는지”를 체크합니다.

예시 상황:

- 스레드 데드락으로 요청 처리가 멈췄지만 프로세스는 살아 있는 경우
- 무한 루프에 빠져 CPU만 100% 쓰고 있는 경우

이때 Liveness Probe가 실패하면:

- kubelet이 컨테이너를 **강제로 죽이고 재시작**합니다.

### 4.2 예시: Spring Boot + Actuator

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: order-service
    spec:
      containers:
        - name: order-service
          image: order-service:latest
          ports:
            - containerPort: 8080
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 3
            failureThreshold: 3
```

> Spring Boot 2.3+에서는 `/actuator/health/liveness`와 `/actuator/health/readiness`를 분리해서 사용하는 것이 권장됩니다.

### 4.3 설정 팁

- `initialDelaySeconds`
  - 컨테이너가 뜨고 **얼마 후부터** Liveness를 체크할지
  - 초기 부팅이 느린 애플리케이션은 충분히 크게 잡기
- `failureThreshold`
  - 몇 번 연속 실패하면 “죽었다”고 볼지
- 너무 aggressive하게 설정하면:
  - 잠깐 느려진 것만으로 계속 재시작되는 “리스타트 루프”가 생길 수 있으니 주의

---

## 5. Readiness Probe: “준비 안 됐으면 트래픽 보내지 마”

### 5.1 개념

**Readiness Probe**는 컨테이너가 **요청을 처리할 준비가 되었는지**를 체크합니다.

시나리오:

- 앱은 살아 있지만
  - DB 마이그레이션 중
  - 캐시 워밍업 중
  - 의존 서비스가 아직 뜨지 않음

이때 Readiness Probe가 실패하면:

- 해당 Pod는 **Service의 Endpoints에서 제외**됩니다.
- 즉, 외부 트래픽은 **준비된 Pod에만** 분배됩니다.

### 5.2 예시: Spring Boot Readiness

```yaml
readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
  timeoutSeconds: 2
  failureThreshold: 3
```

- `/readiness` 엔드포인트에서
  - DB, Redis, 외부 API 상태 등을 종합해서 `UP/DOWN`을 판단하도록 구성하는 것이 일반적입니다.

### 5.3 롤링 업데이트와의 관계

롤링 업데이트 시:

1. 새 Pod가 올라옴
2. **Readiness Probe 성공 전까지는** 트래픽이 가지 않음
3. 새 Pod가 준비되면 트래픽 분산 시작
4. 기존 Pod의 Readiness가 실패하면 트래픽에서 제외되고 제거

즉, Readiness Probe는 **무중단 배포(Zero Downtime)에 매우 중요한 역할**을 합니다.

---

## 6. Startup Probe: “부팅 시간은 좀 기다려줘”

### 6.1 개념

어떤 애플리케이션은 부팅이 매우 느립니다.

- Spring Boot + JPA + Flyway 마이그레이션 + 대량 캐시 로딩 …

이때 Liveness/Readiness를 바로 걸어버리면:

- 아직 부팅 중인 상태를 “죽었다”고 오판하고
- 계속해서 컨테이너를 재시작하는 **Restart Loop**가 발생할 수 있습니다.

**Startup Probe**는 이런 문제를 해결하기 위해 도입된 Probe입니다.

- Startup Probe가 성공할 때까지
  - **Liveness/Readiness Probe는 무시**됩니다.

### 6.2 예시

```yaml
startupProbe:
  httpGet:
    path: /actuator/health
    port: 8080
  # 최대 5분(300초)까지 기다리기
  failureThreshold: 30
  periodSeconds: 10
```

- 위 설정은
  - 10초마다 한 번씩 체크
  - 최대 30번까지 실패 허용 → 최대 300초(5분)까지 부팅을 기다려줌
- Startup Probe가 성공하면 그때부터 Liveness/Readiness가 활성화됩니다.

---

## 7. 세 Probe를 함께 설계해 보기 (Spring Boot 예시)

아래는 하나의 Deployment에 **Startup + Liveness + Readiness**를 모두 적용한 예시입니다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      containers:
        - name: order-service
          image: order-service:latest
          ports:
            - containerPort: 8080

          # 1) Startup Probe: 부팅이 끝날 때까지 기다림
          startupProbe:
            httpGet:
              path: /actuator/health
              port: 8080
            failureThreshold: 30
            periodSeconds: 10

          # 2) Liveness Probe: 살아있는지 체크 (죽으면 재시작)
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 60
            periodSeconds: 15
            timeoutSeconds: 3
            failureThreshold: 3

          # 3) Readiness Probe: 트래픽 받을 준비가 되었는지 체크
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 20
            periodSeconds: 10
            timeoutSeconds: 2
            failureThreshold: 3
```

생각 흐름을 정리해보면:

1. **Startup Probe**
   - “최대 5분 정도까지는 부팅을 기다려 줄게”
2. **Liveness Probe**
   - “부팅이 끝난 뒤에는, 이 애플리케이션이 멈추면 다시 재시작할게”
3. **Readiness Probe**
   - “준비 안 된 Pod에는 트래픽을 보내지 않을게”

---

## 8. 실전에서 자주 하는 실수와 팁

### 8.1 Liveness와 Readiness를 같은 엔드포인트로 쓰는 경우

간단한 서비스에서는 `/health` 하나로 Liveness/Readiness를 동시에 쓰기도 합니다.

하지만 조금만 복잡해져도:

- DB가 잠깐 느려졌는데 Liveness가 실패 → 컨테이너 재시작 루프
- 사실은 “일시적인 의존성 문제”였는데 “죽었다”고 오판

이런 문제가 생기기 쉽습니다.

**권장:**

- **Liveness**: 정말 프로세스가 죽었거나 데드락 상태인지에 집중  
  (예: 스레드 덤프 기반 체크, 내부 상태 플래그)
- **Readiness**: DB/Redis/외부 API 등 의존성 상태까지 고려

Spring Boot Actuator에서는 이미 이를 반영해:

- `/actuator/health/liveness`
- `/actuator/health/readiness`

로 엔드포인트를 분리해 두었습니다.

### 8.2 초기에 너무 aggressive한 설정

처음 Health Check를 도입할 때:

- `initialDelaySeconds`를 너무 작게
- `failureThreshold`를 1로

설정해두면,

- GC나 네트워크 일시적 지연만으로도  
  Pod가 계속 재시작되는 현상을 볼 수 있습니다.

**팁:**

- 운영 환경에 맞게 충분한 여유 시간을 주고
- 실제 장애 로그를 보면서 점진적으로 값을 줄여가는 방식이 안전합니다.

### 8.3 Probe 실패 → 서비스 장애를 디버깅하는 방법

문제가 생기면 다음을 순서대로 확인합니다.

1. `kubectl describe pod`로 **Events**에서 Probe 실패 로그 확인
2. 해당 시점의 애플리케이션 로그/메트릭 (예: Prometheus, Pinpoint) 확인
3. `/actuator/health/*` 응답 내용을 실제로 Curl/브라우저로 확인

이 과정을 통해:

- Health Check 설정이 너무 엄격한지
- 애플리케이션이 실제로 느린/죽은 상태인지

를 분리해서 보는 연습이 중요합니다.

---

## 마무리

이 글에서는 Kubernetes의 세 가지 Health Check:

- **Startup Probe**: 부팅 시간 보장
- **Liveness Probe**: 죽었는지 감지하고 재시작
- **Readiness Probe**: 트래픽을 받을 준비 여부 판단

를 **컨테이너 생명주기 관점**에서 정리하고, Spring Boot와 함께 쓰는 예시를 살펴봤습니다.

**핵심 정리:**

1. Startup Probe로 **부팅 시간**을 넉넉히 보장해주고
2. Liveness Probe로 **데드락/무한루프 등 치명적 상태**를 감지해 재시작하며
3. Readiness Probe로 **의존성 상태를 반영해 트래픽 라우팅**을 제어한다.

Health Check는 단순한 옵션 설정이 아니라,  
**애플리케이션 생명주기와 장애 시나리오를 어떻게 설계할지에 대한 철학**이기도 합니다.

다음 글에서는 런타임 인프라 레벨에서 한 발 더 올라가서,  
**Strangler(스트랭글러) 패턴으로 기존 모놀리스를 점진적으로 MSA 아키텍처로 분리해 나가는 방법**을 정리해보겠습니다. 🚀

