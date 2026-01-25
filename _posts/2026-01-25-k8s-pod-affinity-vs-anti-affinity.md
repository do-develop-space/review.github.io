---
layout: post
title: "Kubernetes Pod Affinity vs AntiAffinity: 다중 서버에서 어떤 전략을 선택할까?"
date: 2026-01-25
categories: [kubernetes, scheduling, architecture]
tags: [Kubernetes, PodAffinity, PodAntiAffinity, 스케줄링, 고가용성, 장애내성, 노드분산]
---

이전 글에서는 Spring Boot 3.x / Java 21 애플리케이션을 Pinpoint APM으로 모니터링하는 방법을 정리했습니다.  
이번 글에서는 **Kubernetes Pod Affinity / Pod AntiAffinity**를 공부하는 느낌으로 정리해보고,  
특히 **다중 서버(Worker Node 여러 개)** 환경에서 어떤 전략을 더 선호해야 하는지 생각해보겠습니다.

---

## 1. Pod Affinity / AntiAffinity 한 줄 정의

먼저 개념을 아주 간단히 요약하면:

- **Pod Affinity**  
  → *“이 Pod는 **저 Pod와 같은 노드**에 배치됐으면 좋겠다”*

- **Pod AntiAffinity**  
  → *“이 Pod는 **저 Pod와 다른 노드**에 배치됐으면 좋겠다”*

Kubernetes 스케줄러에게 **"같이 붙여줘"**(Affinity) 혹은 **"떼어놔줘"**(AntiAffinity)를 힌트로 주는 기능입니다.

---

## 2. 실제 YAML로 보는 Affinity / AntiAffinity

### 2.1 Pod Affinity 예시 (같은 노드에 붙이기)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      affinity:
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app
                    operator: In
                    values:
                      - cache
              topologyKey: "kubernetes.io/hostname"
      containers:
        - name: my-app
          image: my-app:latest
```

- `podAffinity` → `app=cache` 라벨을 가진 Pod가 있는 **같은 노드(hostname)** 에 스케줄해 달라는 의미
- `requiredDuringSchedulingIgnoredDuringExecution` → 요구사항을 **강하게** 적용 (못 맞추면 스케줄링 실패)

### 2.2 Pod AntiAffinity 예시 (노드에 분산 배치)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api-server
  template:
    metadata:
      labels:
        app: api-server
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app
                    operator: In
                    values:
                      - api-server
              topologyKey: "kubernetes.io/hostname"
      containers:
        - name: api-server
          image: api-server:latest
```

- `podAntiAffinity` → 동일 라벨(`app=api-server`)을 가진 Pod끼리는 **같은 노드에 모이지 않게 해달라**는 의미
- 결과: Replica 3개라면 **가능한 한 서로 다른 Worker Node**에 각각 배치

### 2.3 Node Affinity와의 관계 (짧게 정리)

공부하면서 정리해보면, Affinity 계열은 크게 두 축으로 나뉩니다.  
이 구조는 아래 글의 정리가 깔끔해서 참고했습니다. [`출처`](https://nayoungs.tistory.com/entry/Kubernetes-k8s-%EC%96%B4%ED%94%BC%EB%8B%88%ED%8B%B0Affinity%EC%99%80-%EC%95%88%ED%8B%B0-%EC%96%B4%ED%94%BC%EB%8B%88%ED%8B%B0Anti-Affinity)

- **Node Affinity**
  - `pod.spec.affinity.nodeAffinity`
  - **노드의 레이블**을 기준으로 “어느 노드에 스케줄링 가능한지”를 제한
  - nodeSelector의 업그레이드 버전 느낌

- **Pod Affinity / Pod AntiAffinity**
  - `pod.spec.affinity.podAffinity`
  - `pod.spec.affinity.podAntiAffinity`
  - **이미 떠 있는 Pod의 라벨/네임스페이스**를 기준으로  
    “같은 노드에 붙일지 / 다른 노드로 분리할지”를 결정

그리고 세 가지 모두에서 등장하는 공통 개념이 있습니다:

- `requiredDuringSchedulingIgnoredDuringExecution`  
  → **Hard 조건**: 만족 못 하면 아예 스케줄링 실패
- `preferredDuringSchedulingIgnoredDuringExecution`  
  → **Soft 조건**: 최대한 맞추려고 시도하지만, 안 되면 무시하고 배치

Pod Affinity/AntiAffinity 쪽에는 **`topologyKey`** 라는 필드가 추가로 있는데,

- `topologyKey: "kubernetes.io/hostname"` → 같은/다른 **노드 단위**로 묶거나 분리
- 다른 키(예: AZ 레이블)를 쓰면 **가용 영역 단위**로 묶거나 분리하는 것도 가능

---

## 3. 언제 Pod Affinity를 쓰는가?

실제로는 **기본값으로 잘 쓰진 않고, 특정한 이유가 있을 때만** Pod Affinity를 사용합니다.

대표적인 케이스 몇 가지를 정리해보면:

### 3.1 같은 노드에 있는 사이드카/에이전트와 “동거”해야 할 때

예를 들어:

- 특정 Node에만 떠 있는 **로컬 캐시 서버** 또는
- 파일 시스템을 공유하는 **에이전트/데몬셋 Pod**와 함께 있어야 할 때

이럴 때는 해당 Pod와 **같은 Node에 배치**하고 싶을 수 있습니다.

```yaml
affinity:
  podAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
            - key: role
              operator: In
              values:
                - cache-node
        topologyKey: "kubernetes.io/hostname"
```

- `role=cache-node` 라벨을 가진 Pod가 있는 노드에만 현재 Pod를 스케줄링

### 3.2 로컬 디스크/로컬 캐시를 공유해야 할 때

- 노드의 **로컬 SSD**에 캐시 파일을 쌓고,
- 같은 노드에 있는 다른 Pod들이 그 데이터를 재사용해야 할 때

이 경우에도 **같은 노드로 Affinity**를 주는 것이 유리할 수 있습니다.

다만, 이런 구조는:

- 노드 장애 시 해당 노드의 데이터가 통째로 날아가므로
- **내구성/가용성 측면에서 Trade-off**를 감수해야 합니다.

### 3.3 성능 최적화를 위해 노드 내부 통신을 선호할 때

- 매우 빈번한 RPC 호출이 발생하는 두 컴포넌트가 있을 때
- 네트워크 홉(노드 간 통신)을 줄이고 싶으면 같은 노드에 붙여 둘 수도 있습니다.

하지만 Kubernetes + 클라우드 환경에서는:

- 노드 간 네트워크가 충분히 빠른 경우가 많기 때문에
- 이 정도 이유만으로 Affinity를 쓰기보다는,  
  **먼저 Horizontal Scaling, 리소스 튜닝, 캐싱 등을 검토**하는 것이 일반적입니다.

---

## 4. 언제 Pod AntiAffinity를 쓰는가?

다중 서버(Worker Node가 여러 개) 환경에서 **가장 자주 쓰는 쪽은 Pod AntiAffinity**입니다.

### 4.1 고가용성과 장애 내성

예를 들어 `replicas: 3`인 API 서버가 있다고 가정해 보겠습니다.

- **AntiAffinity가 없는 경우:**
  - 스케줄러가 편한 대로 배치하다 보면,  
    **Replica 3개가 모두 같은 노드에 몰릴 수도** 있습니다.
  - 이 노드 하나가 죽으면? → API 서버 3개가 한 번에 사라짐

- **AntiAffinity를 사용하는 경우:**
  - 가능한 한 **각 Replica가 서로 다른 노드**에 배치
  - 노드 하나가 죽어도 나머지 노드의 Pod들로 서비스 유지 가능

```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
            - key: app
              operator: In
              values:
                - api-server
        topologyKey: "kubernetes.io/hostname"
```

### 4.2 리소스 사용 분산

- 같은 타입의 Pod가 한 노드에 몰리면:
  - 그 노드만 CPU/메모리/네트워크가 바빠지고
  - 나머지 노드는 한가한 **“Hot Node” 현상**이 생깁니다.

- AntiAffinity를 적용하면:
  - Pod가 노드 전체에 적절히 퍼져서
  - 리소스 사용이 고르게 분산되고,  
    **예상치 못한 노드 과부하**를 줄일 수 있습니다.

---

## 5. 다중 서버에서 무엇을 더 선호해야 할까?

질문을 다시 적어보면:

> **“다중 서버로 배포될 때, Pod Affinity와 Pod AntiAffinity 중 어떤 환경을 더 선호하나?”**

개인적인 정리와 실무 경험을 섞어서 답을 정리하면:

### 5.1 기본값: AntiAffinity를 선호

다중 서버(Worker Node 여러 개)를 가지고 있고,  
서비스의 **가용성/장애 내성이 중요하다면** 다음처럼 생각하는 게 자연스럽습니다.

- **Stateless 서비스 (웹/API 서버, 백엔드 비즈니스 서비스)**:
  - → **Pod AntiAffinity를 기본 전략으로 두는 것이 안전**
  - 이유:
    - 노드 하나가 죽어도 전체 서비스가 같이 죽지 않도록
    - 장애 영역(failure domain)을 **노드 단위로 쪼개는 효과**

즉, "한 노드가 죽어도 전체가 죽지 않게 하자"라는 관점에서 보면  
**동일한 Pod Replica는 가능한 한 서로 다른 노드에 흩어져 있어야 합니다.**

### 5.2 Affinity는 “특수한 이유가 있을 때만”

반대로, **Pod Affinity는 기본값으로 쓰기보다는 예외적으로** 사용하는 게 좋습니다.

- 특정 데몬셋/캐시/에이전트와 꼭 붙어 있어야 할 때
- 로컬 디스크/스페셜 하드웨어(GPU, SSD 등)를 공유해야 할 때
- 네트워크 특성상 특정 노드에만 접근 가능한 리소스가 있을 때

위에 참고한 글에서도 예시로 많이 나오는 패턴이 하나 있습니다. [`출처`](https://nayoungs.tistory.com/entry/Kubernetes-k8s-%EC%96%B4%ED%94%BC%EB%8B%88%ED%8B%B0Affinity%EC%99%80-%EC%95%88%ED%8B%B0-%EC%96%B4%ED%94%BC%EB%8B%88%ED%8B%B0Anti-Affinity)

- **같은 컨트롤러(ReplicaSet/Deployment)에서 생성된 Pod들끼리는 AntiAffinity로 서로 다른 노드에 분산**하고,
- **Web과 DB와 같이 강하게 연관된 Pod들은 Affinity로 같은 노드에 붙이는** 식의 조합입니다.

즉:

- **“같은 역할의 Pod는 서로 떨어뜨리고, 서로 의존하는 Pod는 붙인다”**  
  라는 식으로 Affinity/AntiAffinity를 조합하는 패턴이 실무에서도 자주 쓰입니다.

이런 경우를 제외하면,

- Affinity로 Pod를 한 노드에 몰아두는 것은
  - 장애 시 리스크를 키우고
  - 리소스 사용도 한쪽으로 쏠리게 만들 수 있습니다.

---

## 6. required vs preferred: 너무 강하게 걸지 말기

Affinity/AntiAffinity에는 두 가지 모드가 있습니다.

- `requiredDuringSchedulingIgnoredDuringExecution`
  - **필수 조건**: 만족하지 못하면 스케줄링 자체가 실패
- `preferredDuringSchedulingIgnoredDuringExecution`
  - **선호 조건**: 되도록 맞춰보되, 안 되면 무시하고 다른 노드에라도 배치

실무 팁:

- 클러스터 노드 수가 적거나, 리소스가 빡빡한 환경이라면
  - 처음부터 `required...`로 강하게 거는 것보다  
  - **`preferred...`로 완화된 정책**을 먼저 적용해 보는 것도 방법입니다.

```yaml
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
              - key: app
                operator: In
                values:
                  - api-server
          topologyKey: "kubernetes.io/hostname"
```

- 이렇게 하면 스케줄러가 "가능하면 분산 배치"를 시도하지만,  
  노드 상황이 너무 안 좋아서 조건을 못 맞추면 **일단 어디든 배치**할 수 있습니다.

---

## 7. 정리

**한 문장 요약:**

- **장애 내성과 고가용성을 우선하는 다중 서버 환경이라면 → Pod AntiAffinity가 기본값에 가깝고,  
  Pod Affinity는 특수한 이유가 있을 때만 신중하게 쓴다.**

정리하면:

1. **Pod Affinity**
   - 특정 Pod/에이전트/캐시와 같은 노드에 두고 싶을 때
   - 로컬 디스크, 특수 하드웨어, 성능 최적화 등 *특별한 요구*가 있을 때만 사용
2. **Pod AntiAffinity**
   - Replica들을 서로 다른 노드에 분산해 **장애 내성**을 높이고 싶을 때
   - Hot Node를 피하고 리소스를 고르게 쓰고 싶을 때
3. **required vs preferred**
   - `required...`는 스케줄링 실패 리스크가 크니,  
     처음엔 `preferred...`로 완화된 전략부터 적용하는 것도 좋다.

다음 글에서는 이 Affinity/AntiAffinity와도 연관된 주제인  
**Kubernetes Health Check (Liveness / Readiness / Startup Probe)**를 정리해보겠습니다. 🚀



