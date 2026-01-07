---
layout: post
title: "Kubernetes Master Node Pod의 Tolerations 설정: Worker Node 메모리 보호"
date: 2026-01-07
categories: [kubernetes, devops]
tags: [Kubernetes, Taint, Toleration, ControlPlane, MasterNode, WorkerNode, 메모리보호, 리소스관리]
---

# Kubernetes Master Node Pod의 Tolerations 설정: Worker Node 메모리 보호

이전 글에서 워커 노드에 Pod를 배포했는데도 마스터 노드의 메모리가 증가하는 문제를 다뤘습니다. 이번 글에서는 반대로 **Master Node에 있는 Pod들(Control Plane 컴포넌트)이 Worker Node에 스케줄링되어 Worker Node의 메모리를 점유하는 문제**를 해결하는 방법을 정리해보겠습니다.

Master Node에 있는 Pod들이 Master Node에 만들어지긴 하지만, Toleration 설정이 없으면 Worker Node에도 스케줄링될 수 있습니다. 이 경우 Worker Node의 메모리가 Control Plane 컴포넌트에 의해 사용되어 애플리케이션 Pod 실행에 필요한 리소스가 부족해질 수 있습니다. **Taint와 Toleration**을 활용하여 Master Node에 있는 Pod들이 Worker Node에 배포되지 않도록 설정하는 방법을 살펴보겠습니다.

---

## 1. 문제 상황

### 1.1 Master Node Pod가 Worker Node에 스케줄링되는 문제

**발생 가능한 시나리오:**

Master Node에 있는 Pod들(Control Plane 컴포넌트)이 Master Node에 만들어지긴 하지만, Toleration 설정이 없으면 Worker Node에도 스케줄링될 수 있습니다.

```
Master Node (Control Plane)
├── kube-apiserver
├── kube-controller-manager (Master Node에 배포됨)
├── kube-scheduler
└── etcd

Worker Node
├── Application Pod 1
├── Application Pod 2
└── kube-controller-manager (❌ Worker Node에도 스케줄링되어 메모리 점유)
    └── Worker Node 메모리 사용: 512Mi
```

**문제점:**
- Master Node에 있는 Pod들이 Worker Node에도 스케줄링될 수 있음
- Worker Node의 메모리와 리소스가 Control Plane 컴포넌트에 의해 점유됨
- Worker Node는 애플리케이션 Pod 실행에 집중해야 하는데, Control Plane이 메모리를 사용
- Master Node의 리소스가 부족하거나 스케줄러가 Worker Node를 선택할 때 Worker Node로 스케줄링될 수 있음

### 1.2 Worker Node 메모리 보호 필요성

**Worker Node의 역할:**
- 애플리케이션 Pod 실행
- 사용자 트래픽 처리
- 비즈니스 로직 실행

**Master Node에 있는 Pod들이 Worker Node에 스케줄링되면:**
- Worker Node의 메모리와 CPU가 Control Plane 컴포넌트에 의해 점유됨
- 애플리케이션 Pod 실행에 필요한 리소스 감소
- Worker Node의 메모리 부족으로 인한 성능 저하 및 안정성 문제 발생 가능

---

## 2. Taint와 Toleration 개념

### 2.1 Taint란?

**Taint(얽힘) = "접근 금지 표지"**
- 노드에 설정하는 라벨 같은 것
- "일반 Pod는 이 노드에 오지 마세요"라는 방어막 역할
- Master Node를 보호하기 위해 자동으로 설정됨

**핵심 개념:**
- **노드의 방어막 역할**: 노드에 Taint가 있으면 일반 Pod는 접근 불가
- **노드에 붙는 설정**: 노드 레벨에서 설정되는 제한

**Taint 구성 요소:**
- `key`: Taint의 이름
- `value`: Taint의 값 (선택적)
- `effect`: Taint의 효과
  - `NoSchedule`: 새로운 Pod 스케줄링 차단 (이미 실행 중인 Pod는 유지)
  - `PreferNoSchedule`: 가능하면 스케줄링 차단 (부득이하면 허용)
  - `NoExecute`: 새로운 Pod는 스케줄링 차단 + 이미 실행 중인 Pod도 제거 가능

**NoSchedule의 의미:**
- "새 Pod는 막지만, Toleration이 있으면 예외 허용"
- 이미 실행 중인 Pod는 그대로 유지
- Toleration이 없는 새 Pod만 스케줄링 차단

**Master Node의 기본 Taint:**

```bash
# Master Node의 Taint 확인
kubectl describe node <master-node-name> | grep Taint

# 출력 예시:
# Taints: node-role.kubernetes.io/control-plane:NoSchedule
```

**Master Node의 control-plane Taint:**
- Kubernetes가 Master Node에 자동으로 설정
- 목적: 일반 Pod가 Master Node에 스케줄링되지 않도록
- 효과: Master Node 리소스를 Control Plane 컴포넌트 전용으로 보호

### 2.2 Toleration이란?

**Toleration(관용) = "면역 패스"**
- Pod에 설정하는 허가증 같은 것
- "이 Pod는 Taint를 무시하고 스케줄링할 수 있어요"라는 면역력 역할
- Taint와 Toleration이 매칭되어야 Pod가 노드에 배포됨

**핵심 개념:**
- **Pod의 면역력 역할**: Pod에 Toleration이 있으면 Taint가 있는 노드에도 접근 가능
- **Pod에 붙는 설정**: Pod 스펙에 설정되는 허가증

**Toleration 구성 요소:**
- `key`: 허용할 Taint의 key
- `operator`: 매칭 연산자 (`Exists` 또는 `Equal`)
  - `Exists`: Taint의 value와 상관없이 key만 일치하면 허용
  - `Equal`: Taint의 key와 value가 모두 일치해야 허용
- `value`: Taint의 value (operator가 `Equal`일 때)
- `effect`: 허용할 Taint의 effect

### 2.3 실제 비유

**회사 건물 비유:**

```
🏢 Master Node (회사 건물)
   🚫 Taint: "일반 직원 출입 금지"

👤 일반 앱 Pod (baro-auth)
   ❌ Toleration 없음
   → Master Node에 접근 불가 ❌

👔 Control Plane Pod (kube-controller-manager)
   ✅ Toleration 있음 (master node taint 허용)
   → Master Node에 접근 가능 ✅
```

**동작 방식:**
1. Master Node에 `node-role.kubernetes.io/control-plane:NoSchedule` Taint가 설정됨
2. 일반 앱 Pod(baro-auth)는 Toleration이 없어서 Master Node에 스케줄링 불가
3. Control Plane Pod(kube-controller-manager)는 Toleration이 있어서 Master Node에 스케줄링 가능
4. 결과: Master Node는 Control Plane 컴포넌트만 실행, 일반 앱은 Worker Node에만 배포

---

## 3. Master Node Pod에 Toleration 설정

### 3.1 Control Plane 컴포넌트 Toleration 설정

**kube-controller-manager, kube-scheduler 등에 Toleration 추가:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kube-controller-manager
  namespace: kube-system
spec:
  template:
    spec:
      tolerations:
        # Master node taint 허용 (master node 메모리 보호를 위한 taint가 설정된 경우)
        - key: node-role.kubernetes.io/control-plane
          operator: Exists
          effect: NoSchedule
      containers:
      - name: kube-controller-manager
        image: k8s.gcr.io/kube-controller-manager:v1.28.0
        # ...
```

**설정 설명:**
- `key: node-role.kubernetes.io/control-plane`: Master Node의 기본 Taint key
- `operator: Exists`: Taint의 value와 상관없이 key만 일치하면 허용
- `effect: NoSchedule`: `NoSchedule` effect를 가진 Taint 허용

### 3.2 완전한 예시: kube-controller-manager

**kube-controller-manager Deployment:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kube-controller-manager
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      component: kube-controller-manager
  template:
    metadata:
      labels:
        component: kube-controller-manager
    spec:
      tolerations:
        # Master node taint 허용 (master node 메모리 보호를 위한 taint가 설정된 경우)
        - key: node-role.kubernetes.io/control-plane
          operator: Exists
          effect: NoSchedule
        # Legacy master taint도 허용 (구버전 호환)
        - key: node-role.kubernetes.io/master
          operator: Exists
          effect: NoSchedule
      nodeSelector:
        node-role.kubernetes.io/control-plane: ""
      containers:
      - name: kube-controller-manager
        image: k8s.gcr.io/kube-controller-manager:v1.28.0
        command:
        - kube-controller-manager
        - --kubeconfig=/etc/kubernetes/controller-manager.conf
        - --bind-address=127.0.0.1
        - --leader-elect=true
        resources:
          requests:
            cpu: 200m
            memory: 256Mi
          limits:
            cpu: 500m
            memory: 512Mi
```

### 3.3 kube-scheduler Toleration 설정

**kube-scheduler Deployment:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kube-scheduler
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      component: kube-scheduler
  template:
    metadata:
      labels:
        component: kube-scheduler
    spec:
      tolerations:
        # Master node taint 허용
        - key: node-role.kubernetes.io/control-plane
          operator: Exists
          effect: NoSchedule
        - key: node-role.kubernetes.io/master
          operator: Exists
          effect: NoSchedule
      nodeSelector:
        node-role.kubernetes.io/control-plane: ""
      containers:
      - name: kube-scheduler
        image: k8s.gcr.io/kube-scheduler:v1.28.0
        command:
        - kube-scheduler
        - --kubeconfig=/etc/kubernetes/scheduler.conf
        - --bind-address=127.0.0.1
        - --leader-elect=true
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 200m
            memory: 256Mi
```

---

## 4. Worker Node 메모리 보호 효과

### 4.1 Toleration 설정 전후 비교

**설정 전:**

```
Master Node
├── kube-apiserver
├── kube-controller-manager
└── kube-scheduler

Worker Node
├── Application Pod 1
├── Application Pod 2
└── kube-controller-manager (❌ Worker Node에 배포됨)
    └── Worker Node 메모리 사용: 512Mi
```

**설정 후:**

```
Master Node
├── kube-apiserver
├── kube-controller-manager (✅ Master Node에만 배포)
└── kube-scheduler

Worker Node
├── Application Pod 1
└── Application Pod 2
    └── Worker Node 메모리: Control Plane 사용 없음 ✅
```

### 4.2 리소스 사용량 비교

**Worker Node 메모리 사용량:**

| 구분 | Toleration 설정 전 | Toleration 설정 후 |
|------|-------------------|-------------------|
| **Control Plane 사용** | 512Mi ~ 1Gi | 0Mi |
| **애플리케이션 Pod 사용** | 제한적 | 전체 사용 가능 |
| **여유 메모리** | 감소 | 증가 |

**효과:**
- Worker Node의 메모리가 Control Plane에 사용되지 않음
- 애플리케이션 Pod 실행에 더 많은 리소스 사용 가능
- Worker Node의 안정성 향상

---

## 5. Taint 확인 및 설정

### 5.1 Master Node Taint 확인

**현재 Taint 확인:**

```bash
# Master Node의 Taint 확인
kubectl describe node <master-node-name> | grep Taint

# 출력 예시:
# Taints: node-role.kubernetes.io/control-plane:NoSchedule
```

**모든 노드의 Taint 확인:**

```bash
# 모든 노드의 Taint 확인
kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints
```

### 5.2 Master Node에 Taint 추가 (수동 설정)

**Master Node에 Taint 추가:**

```bash
# Master Node에 Taint 추가
kubectl taint nodes <master-node-name> \
  node-role.kubernetes.io/control-plane=:NoSchedule

# Legacy master taint 추가 (구버전 호환)
kubectl taint nodes <master-node-name> \
  node-role.kubernetes.io/master=:NoSchedule
```

**Taint 제거:**

```bash
# Taint 제거
kubectl taint nodes <master-node-name> \
  node-role.kubernetes.io/control-plane:NoSchedule-
```

### 5.3 Worker Node에 Taint 추가 (선택적)

**Worker Node 메모리 보호를 위한 추가 Taint:**

```bash
# Worker Node에 애플리케이션 전용 Taint 추가
kubectl taint nodes <worker-node-name> \
  workload-type=application:NoSchedule

# Control Plane Pod는 이 Taint를 tolerate하지 않음
# → Worker Node에 Control Plane Pod 배포 방지
```

**애플리케이션 Pod에 Toleration 추가:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    spec:
      tolerations:
        - key: workload-type
          operator: Equal
          value: application
          effect: NoSchedule
      containers:
      - name: app
        image: my-app:latest
```

---

## 6. 실전 예시: Control Plane 컴포넌트 설정

### 6.1 kube-controller-manager 완전한 설정

**Static Pod 방식 (kubeadm):**

```yaml
# /etc/kubernetes/manifests/kube-controller-manager.yaml
apiVersion: v1
kind: Pod
metadata:
  name: kube-controller-manager
  namespace: kube-system
spec:
  tolerations:
    - key: node-role.kubernetes.io/control-plane
      operator: Exists
      effect: NoSchedule
    - key: node-role.kubernetes.io/master
      operator: Exists
      effect: NoSchedule
  nodeSelector:
    node-role.kubernetes.io/control-plane: ""
  containers:
  - name: kube-controller-manager
    image: k8s.gcr.io/kube-controller-manager:v1.28.0
    command:
    - kube-controller-manager
    - --kubeconfig=/etc/kubernetes/controller-manager.conf
    - --bind-address=127.0.0.1
    - --leader-elect=true
    resources:
      requests:
        cpu: 200m
        memory: 256Mi
      limits:
        cpu: 500m
        memory: 512Mi
```

### 6.2 kube-scheduler 완전한 설정

**Static Pod 방식 (kubeadm):**

```yaml
# /etc/kubernetes/manifests/kube-scheduler.yaml
apiVersion: v1
kind: Pod
metadata:
  name: kube-scheduler
  namespace: kube-system
spec:
  tolerations:
    - key: node-role.kubernetes.io/control-plane
      operator: Exists
      effect: NoSchedule
    - key: node-role.kubernetes.io/master
      operator: Exists
      effect: NoSchedule
  nodeSelector:
    node-role.kubernetes.io/control-plane: ""
  containers:
  - name: kube-scheduler
    image: k8s.gcr.io/kube-scheduler:v1.28.0
    command:
    - kube-scheduler
    - --kubeconfig=/etc/kubernetes/scheduler.conf
    - --bind-address=127.0.0.1
    - --leader-elect=true
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 200m
        memory: 256Mi
```

### 6.3 etcd Toleration 설정

**etcd Pod 설정:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: etcd
  namespace: kube-system
spec:
  tolerations:
    - key: node-role.kubernetes.io/control-plane
      operator: Exists
      effect: NoSchedule
    - key: node-role.kubernetes.io/master
      operator: Exists
      effect: NoSchedule
  nodeSelector:
    node-role.kubernetes.io/control-plane: ""
  containers:
  - name: etcd
    image: k8s.gcr.io/etcd:3.5.9-0
    command:
    - etcd
    - --data-dir=/var/lib/etcd
    - --listen-client-urls=https://127.0.0.1:2379
    - --advertise-client-urls=https://127.0.0.1:2379
    resources:
      requests:
        cpu: 200m
        memory: 256Mi
      limits:
        cpu: 500m
        memory: 512Mi
```

---

## 7. Toleration 설정 확인

### 7.1 Pod Toleration 확인

**특정 Pod의 Toleration 확인:**

```bash
# Pod의 Toleration 확인
kubectl get pod <pod-name> -n kube-system -o yaml | grep -A 10 tolerations

# 또는 describe로 확인
kubectl describe pod <pod-name> -n kube-system | grep -A 5 Tolerations
```

**출력 예시:**

```yaml
Tolerations:
  node-role.kubernetes.io/control-plane:NoSchedule
  node-role.kubernetes.io/master:NoSchedule
```

### 7.2 노드별 Pod 배포 확인

**Master Node에 배포된 Pod 확인:**

```bash
# Master Node에 배포된 Pod 확인
kubectl get pods -n kube-system -o wide --field-selector spec.nodeName=<master-node-name>

# Control Plane 컴포넌트만 확인
kubectl get pods -n kube-system -o wide | grep -E "controller-manager|scheduler|etcd"
```

**Worker Node에 Control Plane Pod가 없는지 확인:**

```bash
# Worker Node에 Control Plane Pod가 배포되지 않았는지 확인
kubectl get pods -n kube-system -o wide --field-selector spec.nodeName=<worker-node-name> | grep -E "controller-manager|scheduler|etcd"

# 결과가 없으면 정상 ✅
```

---

## 8. Best Practices

### 8.1 Toleration 설정 권장사항

**1. Control Plane 컴포넌트는 항상 Toleration 설정:**
- kube-controller-manager
- kube-scheduler
- etcd
- kube-apiserver (일반적으로 Static Pod로 실행)

**2. Legacy Taint도 함께 허용:**
- `node-role.kubernetes.io/control-plane` (신규)
- `node-role.kubernetes.io/master` (구버전 호환)

**3. nodeSelector와 함께 사용:**
- Toleration만으로는 부족할 수 있음
- `nodeSelector`로 Master Node에만 배포되도록 명시

### 8.2 리소스 제한 설정

**Control Plane 컴포넌트에 리소스 제한 설정:**

```yaml
resources:
  requests:
    cpu: 200m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

**효과:**
- Control Plane이 과도한 리소스를 사용하지 않음
- Master Node의 안정성 향상
- Worker Node 메모리 보호와 함께 사용 시 이중 보호

### 8.3 모니터링

**노드별 리소스 사용량 모니터링:**

```bash
# 노드별 리소스 사용량 확인
kubectl top nodes

# Pod별 리소스 사용량 확인
kubectl top pods -n kube-system

# Control Plane 컴포넌트 리소스 사용량 추적
kubectl top pods -n kube-system | grep -E "controller-manager|scheduler|etcd"
```

**알림 설정:**
- Worker Node에 Control Plane Pod가 배포되면 알림
- Control Plane 컴포넌트의 리소스 사용량이 임계값을 초과하면 알림

---

## 9. 문제 해결

### 9.1 Control Plane Pod가 Worker Node에 배포되는 경우

**원인:**
- Toleration이 설정되지 않음
- Taint가 제거됨
- nodeSelector가 설정되지 않음

**해결:**
1. Toleration 추가
2. Master Node에 Taint 추가
3. nodeSelector 추가

### 9.2 Control Plane Pod가 시작되지 않는 경우

**원인:**
- Toleration이 잘못 설정됨
- Taint key가 일치하지 않음
- nodeSelector가 잘못 설정됨

**해결:**
1. Master Node의 Taint 확인
2. Pod의 Toleration 확인
3. nodeSelector 확인

**디버깅:**

```bash
# Pod 이벤트 확인
kubectl describe pod <pod-name> -n kube-system

# 스케줄링 실패 원인 확인
kubectl get events -n kube-system --sort-by='.lastTimestamp' | grep <pod-name>
```

---

## 마무리

**핵심 개념 정리:**

- **Taint = "접근 금지 표지"**: 노드에 설정하는 방어막, "일반 Pod는 이 노드에 오지 마세요"
- **Toleration = "면역 패스"**: Pod에 설정하는 허가증, "이 Pod는 Taint를 무시하고 스케줄링할 수 있어요"
- **NoSchedule**: "새 Pod는 막되, Toleration 있으면 허용" (이미 실행 중인 Pod는 유지)
- **control-plane taint**: Master Node에 자동으로 설정되는 taint, 일반 Pod 스케줄링 차단

**핵심 포인트:**

- **Master Node에 있는 Pod들(Control Plane 컴포넌트)에 Toleration을 설정하여 Worker Node에 배포되지 않도록 해야 합니다.**
- **Worker Node의 메모리를 보호하고, 애플리케이션 Pod 실행에 집중할 수 있도록 해야 합니다.**
- **Toleration과 nodeSelector를 함께 사용하면 더 안전하게 Pod 배포를 제어할 수 있습니다.**
- **리소스 제한과 모니터링을 통해 Control Plane과 Worker Node의 리소스를 효과적으로 관리할 수 있습니다.**

**최종 정리:**
- **Taint**: 노드에 설정하는 "접근 제한" (노드의 방어막 역할)
- **Toleration**: Pod에 설정하는 "면역 패스" (Pod의 면역력 역할)
- **NoSchedule**: "새 Pod는 막지만, Toleration이 있으면 예외 허용"
- **control-plane taint**: Master Node에 자동으로 설정되는 taint

Master Node Pod에 Toleration을 설정하면, Control Plane 컴포넌트가 Worker Node에 배포되지 않아 Worker Node의 메모리를 보호할 수 있습니다. 이는 이전 글에서 다룬 "워커 노드 Pod 배포 시 마스터 노드 메모리 사용 문제"와 반대 관점에서 리소스를 보호하는 중요한 설정입니다.

다음 글에서는 **Public EC2(t3.medium 4GB) 환경에서 여러 Pod를 운영할 때 발생하는 메모리 부족 문제**를 해결하는 최적화 전략을 정리해볼 예정입니다. 🚀

