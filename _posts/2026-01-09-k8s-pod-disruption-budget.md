---
layout: post
title: "Kubernetes Pod Disruption Budget (PDB): Pod 가용성 보장 전략"
date: 2026-01-09
categories: [kubernetes, devops]
tags: [Kubernetes, PDB, PodDisruptionBudget, 가용성, RollingUpdate, 노드드레이닝, 무중단배포]
---

# Kubernetes Pod Disruption Budget (PDB): Pod 가용성 보장 전략

이전 글에서 Public EC2(t3.medium 4GB) 환경에서 여러 Pod를 운영할 때 메모리 최적화 방법을 다뤘습니다. 이번 글에서는 **Pod Disruption Budget(PDB)**을 통해 계획된 중단 중에도 Pod의 가용성을 보장하는 방법을 정리해보겠습니다.

Kubernetes 클러스터에서 노드 업그레이드, 유지보수, 자동 스케일링 등으로 인해 Pod가 중단될 수 있습니다. PDB를 설정하면 이러한 계획된 중단 중에도 최소한의 Pod가 항상 실행되도록 보장할 수 있습니다.

---

## 1. Pod Disruption Budget이란?

### 1.1 PDB의 개념

**Pod Disruption Budget(PDB):**
- 계획된 중단(Planned Disruption) 중에도 Pod의 가용성을 보장하는 Kubernetes 리소스
- 최소 가용 Pod 수 또는 최대 중단 Pod 수를 지정
- 노드 업그레이드, 유지보수, 자동 스케일링 시 Pod 보호

**계획된 중단 (Planned Disruption):**
- 노드 드레이닝 (Node Draining)
- 노드 업그레이드
- 클러스터 유지보수
- Deployment Rolling Update
- 수동 Pod 삭제

**비계획된 중단 (Unplanned Disruption):**
- 노드 장애
- 네트워크 분할
- 하드웨어 오류
- PDB는 비계획된 중단에는 적용되지 않음

### 1.2 PDB의 필요성

**문제 상황:**

```
Deployment: my-app (replicas: 5)

노드 업그레이드 시작
├── Pod 1 삭제 (새 노드로 재배포)
├── Pod 2 삭제 (새 노드로 재배포)
├── Pod 3 삭제 (새 노드로 재배포)
├── Pod 4 삭제 (새 노드로 재배포)
└── Pod 5 삭제 (새 노드로 재배포)

결과: 모든 Pod가 동시에 중단 → 서비스 중단! ❌
```

**PDB 적용 후:**

```
Deployment: my-app (replicas: 5)
PDB: minAvailable: 3

노드 업그레이드 시작
├── Pod 1 삭제 (새 노드로 재배포) ✅
├── Pod 2 삭제 (새 노드로 재배포) ✅
├── Pod 3 유지 (최소 3개 보장)
├── Pod 4 유지 (최소 3개 보장)
└── Pod 5 유지 (최소 3개 보장)

결과: 최소 3개 Pod 유지 → 서비스 지속! ✅
```

---

## 2. PDB 기본 사용법

### 2.1 PDB 생성

**기본 PDB 예시:**

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-app-pdb
  namespace: default
spec:
  minAvailable: 2  # 최소 2개 Pod는 항상 가용해야 함
  selector:
    matchLabels:
      app: my-app
```

**minAvailable 설정:**
- 절대 숫자: `minAvailable: 2` (최소 2개 Pod)
- 백분율: `minAvailable: 50%` (전체의 50%)

**maxUnavailable 설정 (minAvailable과 둘 중 하나만 사용):**

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-app-pdb
spec:
  maxUnavailable: 1  # 최대 1개 Pod만 중단 가능
  selector:
    matchLabels:
      app: my-app
```

**maxUnavailable 설정:**
- 절대 숫자: `maxUnavailable: 1` (최대 1개 Pod)
- 백분율: `maxUnavailable: 25%` (전체의 25%)

### 2.2 PDB 적용 확인

**PDB 생성:**

```bash
# PDB 생성
kubectl apply -f pod-disruption-budget.yaml

# PDB 확인
kubectl get pdb

# 상세 정보 확인
kubectl describe pdb my-app-pdb
```

**출력 예시:**

```
Name:           my-app-pdb
Namespace:      default
Min available:  2
Selector:       app=my-app
Status:
  Allowed disruptions:  3
  Current:              5
  Desired:              5
  Total:                 5
```

---

## 3. PDB 설정 전략

### 3.1 minAvailable vs maxUnavailable

**minAvailable 사용 시나리오:**
- 최소 가용 Pod 수를 명확히 지정하고 싶을 때
- 예: "최소 3개 Pod는 항상 실행되어야 함"

**maxUnavailable 사용 시나리오:**
- 최대 중단 Pod 수를 제한하고 싶을 때
- 예: "최대 1개 Pod만 중단 가능"

**둘 중 하나만 사용:**
- `minAvailable`과 `maxUnavailable`은 동시에 사용할 수 없음
- 둘 중 하나만 지정해야 함

### 3.2 백분율 vs 절대 숫자

**절대 숫자 사용:**

```yaml
spec:
  minAvailable: 3  # 항상 3개 Pod 유지
```

**장점:**
- 명확하고 예측 가능
- Pod 수가 적을 때 유용

**단점:**
- Pod 수가 변경되면 PDB도 수정 필요

**백분율 사용:**

```yaml
spec:
  minAvailable: 50%  # 전체의 50% 유지
```

**장점:**
- Pod 수가 변경되어도 자동으로 조정
- HPA와 함께 사용 시 유용

**단점:**
- Pod 수가 적을 때 부정확할 수 있음 (예: 1개 Pod, 50% = 0.5개)

### 3.3 실전 예시

**프로덕션 환경:**

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: production-app-pdb
  namespace: production
spec:
  minAvailable: 2  # 최소 2개 Pod 유지
  selector:
    matchLabels:
      app: production-app
      tier: backend
```

**스테이징 환경:**

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: staging-app-pdb
  namespace: staging
spec:
  maxUnavailable: 1  # 최대 1개 Pod만 중단
  selector:
    matchLabels:
      app: staging-app
```

**개발 환경:**

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: dev-app-pdb
  namespace: development
spec:
  minAvailable: 50%  # 전체의 50% 유지
  selector:
    matchLabels:
      app: dev-app
```

---

## 4. PDB 동작 원리

### 4.1 노드 드레이닝 시 PDB 동작

**노드 드레이닝 시작:**

```bash
# 노드 드레이닝 시작
kubectl drain <node-name> --ignore-daemonsets
```

**PDB 동작 과정:**

```
1. 노드 드레이닝 요청
   ↓
2. PDB 확인 (해당 노드의 Pod가 PDB 대상인지)
   ↓
3. PDB 조건 확인 (minAvailable 또는 maxUnavailable)
   ↓
4. 조건 만족 시 Pod 삭제 허용
   조건 불만족 시 Pod 삭제 대기
   ↓
5. 새 Pod가 다른 노드에 배포됨
   ↓
6. 가용 Pod 수가 조건을 만족하면 다음 Pod 삭제
```

**예시:**

```
Deployment: my-app (replicas: 5)
PDB: minAvailable: 3

현재 상태: Pod 5개 모두 실행 중
노드 드레이닝 시작

1. Pod 1 삭제 시도 → 가용 Pod 4개 (≥ 3) → 삭제 허용 ✅
2. Pod 2 삭제 시도 → 가용 Pod 3개 (≥ 3) → 삭제 허용 ✅
3. Pod 3 삭제 시도 → 가용 Pod 2개 (< 3) → 삭제 대기 ⏳
4. Pod 1이 새 노드에서 실행됨 → 가용 Pod 3개 (≥ 3)
5. Pod 3 삭제 허용 ✅
```

### 4.2 Rolling Update 시 PDB 동작

**Deployment Rolling Update:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
```

**PDB와의 상호작용:**

```
Deployment: my-app (replicas: 5)
PDB: minAvailable: 3
RollingUpdate: maxUnavailable: 1

Rolling Update 시작:
1. 새 Pod 1개 생성 (maxSurge: 1)
2. 기존 Pod 1개 삭제 시도
   - PDB 확인: 가용 Pod 5개 (≥ 3) → 삭제 허용 ✅
3. 새 Pod 1개 생성
4. 기존 Pod 1개 삭제 시도
   - PDB 확인: 가용 Pod 4개 (≥ 3) → 삭제 허용 ✅
5. 반복...
```

**중요:**
- Deployment의 `maxUnavailable`과 PDB의 `minAvailable`은 독립적으로 동작
- 둘 다 만족해야 Pod 삭제 가능

---

## 5. PDB 실전 활용

### 5.1 마이크로서비스별 PDB 설정

**주문 서비스 (Order Service):**

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: order-service-pdb
  namespace: order-service
spec:
  minAvailable: 2  # 최소 2개 Pod 유지
  selector:
    matchLabels:
      app: order-service
```

**결제 서비스 (Payment Service):**

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: payment-service-pdb
  namespace: payment-service
spec:
  minAvailable: 1  # 최소 1개 Pod 유지
  selector:
    matchLabels:
      app: payment-service
```

**인증 서비스 (Auth Service):**

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: auth-service-pdb
  namespace: auth-service
spec:
  minAvailable: 50%  # 전체의 50% 유지
  selector:
    matchLabels:
      app: auth-service
```

### 5.2 환경별 PDB 설정

**프로덕션 환경:**

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: production-pdb
  namespace: production
spec:
  minAvailable: 3  # 높은 가용성 요구
  selector:
    matchLabels:
      environment: production
```

**스테이징 환경:**

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: staging-pdb
  namespace: staging
spec:
  minAvailable: 1  # 낮은 가용성 요구
  selector:
    matchLabels:
      environment: staging
```

### 5.3 StatefulSet과 PDB

**StatefulSet PDB 설정:**

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: mysql-pdb
  namespace: database
spec:
  minAvailable: 2  # 최소 2개 Pod 유지 (고가용성)
  selector:
    matchLabels:
      app: mysql
```

**주의사항:**
- StatefulSet은 순서대로 Pod를 삭제/생성
- PDB는 StatefulSet의 순서 보장과 함께 동작

---

## 6. PDB 제한사항 및 주의사항

### 6.1 PDB 제한사항

**1. 비계획된 중단에는 적용 안 됨:**
- 노드 장애, 네트워크 분할 등
- PDB는 계획된 중단에만 적용

**2. Pod가 Ready 상태여야 함:**
- PDB는 Ready Pod만 카운트
- NotReady Pod는 가용 Pod로 간주하지 않음

**3. 단일 Pod에는 적용 불가:**
- replicas: 1인 경우 PDB 설정 불가
- minAvailable: 1 또는 maxUnavailable: 0 설정 불가

**4. DaemonSet에는 적용 불가:**
- DaemonSet Pod는 PDB 대상이 아님

### 6.2 주의사항

**PDB 설정이 너무 엄격한 경우:**

```yaml
spec:
  minAvailable: 4  # replicas: 5인 경우
```

**문제:**
- 노드 드레이닝이 매우 느려짐
- 한 번에 1개 Pod만 삭제 가능
- 업그레이드 시간이 길어짐

**권장:**
- `minAvailable`은 전체 Pod의 50-70% 정도
- 또는 `maxUnavailable: 1-2` 정도

**PDB 설정이 너무 느슨한 경우:**

```yaml
spec:
  maxUnavailable: 4  # replicas: 5인 경우
```

**문제:**
- 대부분의 Pod가 동시에 중단 가능
- 서비스 가용성 저하

**권장:**
- `maxUnavailable`은 전체 Pod의 20-30% 정도

---

## 7. PDB 모니터링

### 7.1 PDB 상태 확인

**PDB 상태 확인:**

```bash
# PDB 목록 확인
kubectl get pdb

# 특정 PDB 상세 정보
kubectl describe pdb my-app-pdb

# PDB 상태를 JSON으로 확인
kubectl get pdb my-app-pdb -o json
```

**상태 정보:**
- `Allowed disruptions`: 현재 허용 가능한 중단 수
- `Current`: 현재 Ready Pod 수
- `Desired`: 원하는 Pod 수
- `Total`: 전체 Pod 수

### 7.2 PDB 이벤트 모니터링

**PDB 관련 이벤트 확인:**

```bash
# PDB 관련 이벤트 확인
kubectl get events --field-selector involvedObject.kind=PodDisruptionBudget

# 특정 네임스페이스의 이벤트
kubectl get events -n production --sort-by='.lastTimestamp'
```

### 7.3 Prometheus 메트릭

**PDB 메트릭 수집:**

```yaml
# kube-state-metrics를 통해 PDB 메트릭 수집
- job_name: 'kube-state-metrics'
  kubernetes_sd_configs:
  - role: pod
    namespaces:
      names:
      - kube-system
  relabel_configs:
  - source_labels: [__meta_kubernetes_pod_name]
    regex: 'kube-state-metrics.*'
    action: keep
```

**Grafana 대시보드:**

```yaml
panels:
- title: "PDB Status"
  targets:
  - expr: |
      kube_poddisruptionbudget_status_current_healthy{namespace="production"}
    legendFormat: "Current Healthy Pods"
  - expr: |
      kube_poddisruptionbudget_status_desired_healthy{namespace="production"}
    legendFormat: "Desired Healthy Pods"
```

---

## 8. Best Practices

### 8.1 PDB 설정 권장사항

**1. 적절한 minAvailable/maxUnavailable 설정:**
- 프로덕션: `minAvailable: 50-70%` 또는 `maxUnavailable: 1-2`
- 스테이징: `minAvailable: 1` 또는 `maxUnavailable: 1`
- 개발: PDB 설정 생략 가능

**2. 환경별 차별화:**
- 프로덕션: 높은 가용성 요구
- 스테이징: 낮은 가용성 요구
- 개발: PDB 불필요

**3. 서비스 중요도에 따른 설정:**
- 핵심 서비스: 높은 minAvailable
- 일반 서비스: 중간 minAvailable
- 배치 작업: PDB 불필요

### 8.2 PDB 설정 체크리스트

**PDB 생성 전 확인:**
- [ ] Deployment/StatefulSet의 replicas 수 확인
- [ ] 서비스의 가용성 요구사항 확인
- [ ] 환경별(프로덕션/스테이징) 차별화 고려
- [ ] minAvailable/maxUnavailable 적절성 검토

**PDB 생성 후 확인:**
- [ ] PDB 상태 확인 (`kubectl get pdb`)
- [ ] 노드 드레이닝 테스트
- [ ] Rolling Update 테스트
- [ ] 모니터링 및 알림 설정

---

## 9. 문제 해결

### 9.1 PDB가 Pod 삭제를 막는 경우

**증상:**
- 노드 드레이닝이 진행되지 않음
- Pod가 삭제되지 않음

**원인:**
- PDB의 minAvailable/maxUnavailable 조건 불만족
- Ready Pod 수가 부족

**해결:**
1. PDB 상태 확인: `kubectl describe pdb <pdb-name>`
2. Ready Pod 수 확인: `kubectl get pods`
3. PDB 설정 조정 (필요시)
4. 수동으로 Pod 삭제 (긴급 시)

### 9.2 PDB가 적용되지 않는 경우

**원인:**
- Pod의 라벨이 PDB selector와 일치하지 않음
- Pod가 Ready 상태가 아님
- 단일 Pod (replicas: 1)

**해결:**
1. Pod 라벨 확인: `kubectl get pods --show-labels`
2. PDB selector 확인: `kubectl describe pdb <pdb-name>`
3. Pod 상태 확인: `kubectl get pods`
4. 라벨 수정 또는 PDB selector 수정

---

## 마무리

**핵심 포인트:**

- **Pod Disruption Budget(PDB)은 계획된 중단 중에도 Pod의 가용성을 보장하는 중요한 리소스입니다.**
- **minAvailable 또는 maxUnavailable을 설정하여 최소 가용 Pod 수를 보장할 수 있습니다.**
- **환경별(프로덕션/스테이징)과 서비스 중요도에 따라 PDB 설정을 차별화해야 합니다.**
- **PDB는 비계획된 중단에는 적용되지 않으며, Ready Pod만 카운트합니다.**

PDB를 적절히 설정하면 노드 업그레이드, 유지보수, Rolling Update 중에도 서비스 가용성을 보장할 수 있습니다. 특히 프로덕션 환경에서는 필수적인 설정입니다.

다음 글에서는 **멱등성 키 기반 동시성 제어: PK 기반 락 vs Named Lock**에 대해 정리해볼 예정입니다. 🚀


