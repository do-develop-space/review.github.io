---
layout: post
title: "Kubernetes 워커 노드 Pod 배포 시 마스터 노드 메모리 사용 문제: Control Plane 관리 오버헤드"
date: 2026-01-04
categories: [kubernetes, troubleshooting]
tags: [Kubernetes, ControlPlane, 마스터노드, 메모리문제, kube-controller-manager, kube-scheduler, 리소스관리]
---

# Kubernetes 워커 노드 Pod 배포 시 마스터 노드 메모리 사용 문제: Control Plane 관리 오버헤드

이전 글에서 Kubernetes Ingress를 활용한 Canary 배포 전략을 다뤘습니다. 이번 글에서는 **워커 노드에 Pod를 배포했는데도 마스터 노드의 메모리 사용량이 계속 증가하는 문제**를 다뤄보겠습니다.

Kubernetes 클러스터에서 마스터 노드(Control Plane)의 메모리 사용량이 계속 증가하는 문제가 발생했습니다. 이를 해결하기 위해 `nodeAffinity`를 설정하여 워커 노드에만 Pod를 배포하도록 했고, Pod는 정상적으로 워커 노드에 배포되었습니다.

하지만 **마스터 노드의 메모리 사용량은 여전히 계속 증가**했습니다. 이 문제는 Pod가 마스터 노드에 배포된 것이 아니라, **Control Plane 컴포넌트가 워커 노드의 Pod를 관리하면서 메모리를 사용**하기 때문입니다.

---

## 1. 문제 상황

### 1.1 초기 문제: 마스터 노드 메모리 사용량 증가

**발생한 증상:**
- 마스터 노드의 메모리 사용량이 계속 증가
- Control Plane 컴포넌트(kube-controller-manager, kube-scheduler 등)의 메모리 사용량 증가
- Pod 수가 많아질수록 마스터 노드 메모리 사용량 증가
- 클러스터 불안정 가능성

**초기 확인:**

```bash
# 마스터 노드의 리소스 사용량 확인
kubectl top node <master-node-name>

# Control Plane 컴포넌트의 메모리 사용량 확인
kubectl top pods -n kube-system

# Pod 배포 위치 확인
kubectl get pods -o wide
```

### 1.2 해결 시도: nodeAffinity 추가

**문제 해결을 위해 추가한 설정:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  template:
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: node-role.kubernetes.io/control-plane
                    operator: DoesNotExist
                  - key: node-role.kubernetes.io/master
                    operator: DoesNotExist
      containers:
      - name: app
        image: my-app:latest
```

**의도:**
- `control-plane` 또는 `master` 라벨이 없는 노드에만 배포
- 마스터 노드에는 배포하지 않음
- 마스터 노드 메모리 사용량 감소 기대

### 1.3 여전히 발생하는 문제

**nodeAffinity 추가 후에도:**
- Pod는 정상적으로 워커 노드에만 배포됨 (nodeAffinity 작동) ✅
- 하지만 마스터 노드의 메모리 사용량은 여전히 계속 증가 ❌
- Control Plane 컴포넌트의 메모리 사용량이 계속 증가
- Pod 수가 증가할수록 마스터 노드 메모리 사용량도 증가

**확인 방법:**

```bash
# Pod가 워커 노드에만 배포되었는지 확인
kubectl get pods -o wide

# 마스터 노드의 리소스 사용량 확인
kubectl top node <master-node-name>

# Control Plane 컴포넌트의 메모리 사용량 확인
kubectl top pods -n kube-system

# kube-controller-manager 메모리 확인
kubectl top pod -n kube-system -l component=kube-controller-manager

# kube-scheduler 메모리 확인
kubectl top pod -n kube-system -l component=kube-scheduler
```

---

## 2. 문제 원인 분석

### 2.1 Control Plane의 역할

**Control Plane 컴포넌트들이 하는 일:**

1. **kube-controller-manager**
   - Deployment, ReplicaSet, StatefulSet 등 관리
   - Pod 상태 모니터링 및 복구
   - 각 리소스마다 Controller가 동작
   - **Pod 수가 많을수록 메모리 사용량 증가**

2. **kube-scheduler**
   - Pod 스케줄링 결정
   - 노드 리소스 모니터링
   - **스케줄링 대상이 많을수록 메모리 사용량 증가**

3. **kube-apiserver**
   - 모든 리소스의 상태를 etcd와 동기화
   - Watch API를 통한 실시간 모니터링
   - **리소스가 많을수록 메모리 사용량 증가**

4. **etcd**
   - 모든 리소스 상태 저장
   - **리소스가 많을수록 메모리 사용량 증가**

### 2.2 왜 워커 노드 Pod가 마스터 노드 메모리를 사용하는가?

**원인:**
- Pod는 워커 노드에 실행되지만, **상태 관리와 모니터링은 Control Plane이 담당**
- Control Plane 컴포넌트들이 모든 Pod의 상태를 추적하고 관리
- Pod 수가 증가하면 Control Plane의 메모리 사용량도 증가

**메모리 사용 패턴:**

```
Pod 수 증가
    ↓
kube-controller-manager: 각 Pod 상태 추적 (메모리 증가)
    ↓
kube-scheduler: 스케줄링 정보 관리 (메모리 증가)
    ↓
kube-apiserver: Watch 연결 및 상태 동기화 (메모리 증가)
    ↓
etcd: 리소스 상태 저장 (메모리 증가)
    ↓
마스터 노드 메모리 사용량 증가
```

### 2.3 실제 메모리 사용 패턴

**실제 측정 예시:**

```bash
# Pod 수에 따른 Control Plane 메모리 사용량
Pod 수: 10개
  - kube-controller-manager: ~200MB
  - kube-scheduler: ~100MB
  - kube-apiserver: ~300MB
  - etcd: ~500MB
  총합: ~1.1GB

Pod 수: 100개
  - kube-controller-manager: ~500MB
  - kube-scheduler: ~200MB
  - kube-apiserver: ~800MB
  - etcd: ~1.5GB
  총합: ~3GB

Pod 수: 1000개
  - kube-controller-manager: ~2GB
  - kube-scheduler: ~500MB
  - kube-apiserver: ~3GB
  - etcd: ~5GB
  총합: ~10.5GB
```

**문제점:**
- Pod는 워커 노드에 실행되지만, Control Plane이 모든 Pod를 관리
- Pod 수가 증가하면 Control Plane 메모리 사용량도 선형적으로 증가
- 마스터 노드의 메모리가 부족하면 클러스터 전체가 불안정해짐

---

## 3. 해결 방법

### 3.1 방법 1: Control Plane 컴포넌트 리소스 제한 설정

**kube-controller-manager 리소스 제한:**

```yaml
# /etc/kubernetes/manifests/kube-controller-manager.yaml
apiVersion: v1
kind: Pod
metadata:
  name: kube-controller-manager
  namespace: kube-system
spec:
  containers:
  - name: kube-controller-manager
    command:
    - kube-controller-manager
    - --leader-elect=true
    - --controllers=*,bootstrapsigner,tokencleaner
    resources:
      requests:
        memory: "200Mi"
        cpu: "100m"
      limits:
        memory: "1Gi"  # 메모리 제한 설정
        cpu: "500m"
```

**kube-scheduler 리소스 제한:**

```yaml
# /etc/kubernetes/manifests/kube-scheduler.yaml
apiVersion: v1
kind: Pod
metadata:
  name: kube-scheduler
  namespace: kube-system
spec:
  containers:
  - name: kube-scheduler
    command:
    - kube-scheduler
    - --leader-elect=true
    resources:
      requests:
        memory: "100Mi"
        cpu: "100m"
      limits:
        memory: "500Mi"  # 메모리 제한 설정
        cpu: "500m"
```

**주의:**
- 리소스 제한이 너무 낮으면 Control Plane이 제대로 동작하지 않을 수 있음
- 실제 Pod 수와 리소스 사용량을 모니터링하여 적절한 값 설정

### 3.2 방법 2: etcd 최적화

**etcd 메모리 최적화:**

```yaml
# etcd 설정 파일 수정
# /etc/kubernetes/manifests/etcd.yaml
apiVersion: v1
kind: Pod
metadata:
  name: etcd
  namespace: kube-system
spec:
  containers:
  - name: etcd
    command:
    - etcd
    - --max-request-bytes=1572864  # 요청 크기 제한
    - --quota-backend-bytes=8589934592  # 백엔드 저장소 크기 제한 (8GB)
    resources:
      requests:
        memory: "1Gi"
        cpu: "500m"
      limits:
        memory: "4Gi"  # etcd 메모리 제한
        cpu: "2000m"
```

**etcd 압축 설정:**

```yaml
# etcd에 압축 설정 추가
- --auto-compaction-mode=periodic
- --auto-compaction-retention=1h  # 1시간마다 압축
```

### 3.3 방법 3: kube-apiserver 최적화

**kube-apiserver 리소스 제한:**

```yaml
# /etc/kubernetes/manifests/kube-apiserver.yaml
apiVersion: v1
kind: Pod
metadata:
  name: kube-apiserver
  namespace: kube-system
spec:
  containers:
  - name: kube-apiserver
    command:
    - kube-apiserver
    - --max-requests-inflight=400  # 동시 요청 수 제한
    - --max-mutating-requests-inflight=200
    resources:
      requests:
        memory: "500Mi"
        cpu: "500m"
      limits:
        memory: "2Gi"  # 메모리 제한
        cpu: "2000m"
```

### 3.4 방법 4: Pod 수 제한 및 리소스 관리

**네임스페이스별 리소스 제한:**

```yaml
# ResourceQuota로 Pod 수 제한
apiVersion: v1
kind: ResourceQuota
metadata:
  name: pod-limit
  namespace: default
spec:
  hard:
    pods: "100"  # 최대 100개 Pod
    requests.memory: "50Gi"
    limits.memory: "100Gi"
```

**LimitRange로 개별 Pod 리소스 제한:**

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: pod-limit-range
  namespace: default
spec:
  limits:
  - default:
      memory: "256Mi"
      cpu: "200m"
    defaultRequest:
      memory: "128Mi"
      cpu: "100m"
    type: Container
```

### 3.5 방법 5: Control Plane 모니터링 및 알림

**Prometheus 메트릭 수집:**

```yaml
# kube-controller-manager 메트릭
- job_name: 'kube-controller-manager'
  kubernetes_sd_configs:
  - role: pod
    namespaces:
      names:
      - kube-system
  relabel_configs:
  - source_labels: [__meta_kubernetes_pod_name]
    regex: 'kube-controller-manager.*'
    action: keep
```

**Grafana 알림 규칙:**

```yaml
groups:
- name: control_plane_memory
  rules:
  - alert: HighControlPlaneMemoryUsage
    expr: container_memory_usage_bytes{pod=~"kube-controller-manager.*|kube-scheduler.*|kube-apiserver.*"} > 1073741824  # 1GB
    for: 5m
    annotations:
      summary: "Control Plane memory usage is high"
      description: "{{ $labels.pod }} is using {{ $value }} bytes of memory"
```

---

## 4. 실전 해결 예시

### 4.1 문제 진단

**1단계: Pod 배포 위치 확인**

```bash
# Pod가 워커 노드에만 배포되었는지 확인
kubectl get pods -o wide

# 마스터 노드에 워크로드 Pod가 있는지 확인
kubectl get pods -o wide --all-namespaces --field-selector spec.nodeName=<master-node-name> | grep -v kube-system
```

**2단계: Control Plane 메모리 사용량 확인**

```bash
# 마스터 노드 전체 메모리 사용량
kubectl top node <master-node-name>

# Control Plane 컴포넌트별 메모리 사용량
kubectl top pods -n kube-system

# kube-controller-manager 메모리
kubectl top pod -n kube-system -l component=kube-controller-manager

# kube-scheduler 메모리
kubectl top pod -n kube-system -l component=kube-scheduler

# kube-apiserver 메모리
kubectl top pod -n kube-system -l component=kube-apiserver

# etcd 메모리
kubectl top pod -n kube-system -l component=etcd
```

**3단계: Pod 수와 메모리 사용량 상관관계 확인**

```bash
# 전체 Pod 수
kubectl get pods --all-namespaces | wc -l

# Control Plane 메모리 사용량과 Pod 수 비교
echo "Pod 수: $(kubectl get pods --all-namespaces | wc -l)"
echo "Controller Manager: $(kubectl top pod -n kube-system -l component=kube-controller-manager --no-headers | awk '{print $2}')"
echo "Scheduler: $(kubectl top pod -n kube-system -l component=kube-scheduler --no-headers | awk '{print $2}')"
echo "API Server: $(kubectl top pod -n kube-system -l component=kube-apiserver --no-headers | awk '{print $2}')"
```

### 4.2 해결 적용

**1단계: Control Plane 컴포넌트 리소스 제한 설정**

```bash
# kube-controller-manager 리소스 제한 추가
sudo vi /etc/kubernetes/manifests/kube-controller-manager.yaml

# resources 섹션 추가
resources:
  requests:
    memory: "200Mi"
    cpu: "100m"
  limits:
    memory: "1Gi"
    cpu: "500m"
```

**2단계: etcd 최적화**

```bash
# etcd 설정 파일 수정
sudo vi /etc/kubernetes/manifests/etcd.yaml

# 압축 설정 추가
- --auto-compaction-mode=periodic
- --auto-compaction-retention=1h

# 리소스 제한 추가
resources:
  requests:
    memory: "1Gi"
    cpu: "500m"
  limits:
    memory: "4Gi"
    cpu: "2000m"
```

**3단계: ResourceQuota 설정**

```yaml
# resource-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: pod-limit
  namespace: default
spec:
  hard:
    pods: "100"  # 네임스페이스당 최대 Pod 수
    requests.memory: "50Gi"
    limits.memory: "100Gi"
```

```bash
kubectl apply -f resource-quota.yaml
```

**4단계: 모니터링 설정**

```yaml
# prometheus-config.yaml
- job_name: 'control-plane'
  kubernetes_sd_configs:
  - role: pod
    namespaces:
      names:
      - kube-system
  relabel_configs:
  - source_labels: [__meta_kubernetes_pod_name]
    regex: 'kube-(controller-manager|scheduler|apiserver|etcd).*'
    action: keep
```

---

## 5. 추가 최적화 방법

### 5.1 Controller Manager 최적화

**불필요한 Controller 비활성화:**

```yaml
# kube-controller-manager.yaml
command:
- kube-controller-manager
- --controllers=*,bootstrapsigner,tokencleaner
- --leader-elect=true
# 특정 Controller 비활성화 (필요한 경우)
- --controllers=-ttl,-bootstrapsigner,-tokencleaner
```

### 5.2 API Server 최적화

**Watch 캐시 크기 조정:**

```yaml
# kube-apiserver.yaml
command:
- kube-apiserver
- --watch-cache=true
- --watch-cache-sizes=#PODS=1000,#NODES=100  # 캐시 크기 제한
```

### 5.3 etcd 압축 및 정리

**수동 압축:**

```bash
# etcd Pod에 접속
kubectl exec -it -n kube-system etcd-<node-name> -- sh

# 압축 실행
ETCDCTL_API=3 etcdctl compact <revision>
ETCDCTL_API=3 etcdctl defrag
```

**자동 압축 설정:**

```yaml
# etcd.yaml
command:
- etcd
- --auto-compaction-mode=periodic
- --auto-compaction-retention=1h  # 1시간마다 압축
- --quota-backend-bytes=8589934592  # 8GB 제한
```

---

## 6. 왜 이런 문제가 발생했는가?

### 6.1 Control Plane의 역할

**Kubernetes 아키텍처:**

```
워커 노드: Pod 실행
    ↓
마스터 노드: Pod 관리 및 모니터링
    ├── kube-controller-manager: Pod 상태 추적
    ├── kube-scheduler: Pod 스케줄링
    ├── kube-apiserver: Pod 상태 동기화
    └── etcd: Pod 상태 저장
```

**핵심:**
- Pod는 워커 노드에서 실행되지만
- **모든 Pod의 상태는 Control Plane이 관리**
- Pod 수가 증가하면 Control Plane 메모리 사용량도 증가

### 6.2 메모리 사용 패턴

**kube-controller-manager:**
- 각 리소스(Deployment, ReplicaSet, Pod 등)마다 Controller 실행
- Pod 수가 증가하면 Controller가 추적해야 할 객체 수 증가
- 메모리 사용량 = O(리소스 수)

**kube-scheduler:**
- 스케줄링 결정을 위한 노드 정보 캐싱
- Pod 수가 많을수록 스케줄링 정보 관리 필요
- 메모리 사용량 = O(노드 수 × Pod 수)

**kube-apiserver:**
- Watch API를 통한 실시간 상태 동기화
- 각 Pod에 대한 Watch 연결 관리
- 메모리 사용량 = O(Watch 연결 수)

**etcd:**
- 모든 리소스 상태 저장
- Pod 수가 증가하면 저장 데이터 증가
- 메모리 사용량 = O(리소스 수)

### 6.3 실제 발생 시나리오

**시나리오 1: Pod 수 급증**

```bash
# 초기: Pod 10개
Control Plane 메모리: ~1GB

# Pod 100개로 증가
Control Plane 메모리: ~3GB

# Pod 1000개로 증가
Control Plane 메모리: ~10GB

# 마스터 노드 메모리 부족 → 클러스터 불안정
```

**시나리오 2: 리소스 제한 없음**

```yaml
# Control Plane 컴포넌트에 리소스 제한이 없는 경우
# → 무제한으로 메모리 사용 가능
# → 마스터 노드 메모리 고갈
```

**시나리오 3: etcd 압축 미설정**

```bash
# etcd에 압축이 설정되지 않은 경우
# → 오래된 데이터가 계속 쌓임
# → etcd 메모리 사용량 지속 증가
```

---

## 7. 예방 및 모니터링

### 7.1 예방 설정

**Control Plane 컴포넌트 리소스 제한 (필수):**

```yaml
# 모든 Control Plane 컴포넌트에 리소스 제한 설정
# kube-controller-manager, kube-scheduler, kube-apiserver, etcd
resources:
  requests:
    memory: "200Mi"
    cpu: "100m"
  limits:
    memory: "1Gi"  # 적절한 제한 설정
    cpu: "500m"
```

**ResourceQuota로 Pod 수 제한:**

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: pod-limit
  namespace: default
spec:
  hard:
    pods: "100"  # 네임스페이스당 최대 Pod 수
```

**etcd 압축 설정:**

```yaml
# etcd.yaml
command:
- etcd
- --auto-compaction-mode=periodic
- --auto-compaction-retention=1h
- --quota-backend-bytes=8589934592  # 8GB 제한
```

### 7.2 모니터링

**Control Plane 메모리 모니터링 스크립트:**

```bash
#!/bin/bash
# control-plane-memory-monitor.sh

echo "=== Control Plane Memory Usage ==="
echo ""

# 전체 Pod 수
POD_COUNT=$(kubectl get pods --all-namespaces --no-headers | wc -l)
echo "Total Pods: $POD_COUNT"
echo ""

# Control Plane 컴포넌트별 메모리
echo "Control Plane Components Memory:"
kubectl top pods -n kube-system -l component=kube-controller-manager --no-headers 2>/dev/null | awk '{print "  Controller Manager: " $3}'
kubectl top pods -n kube-system -l component=kube-scheduler --no-headers 2>/dev/null | awk '{print "  Scheduler: " $3}'
kubectl top pods -n kube-system -l component=kube-apiserver --no-headers 2>/dev/null | awk '{print "  API Server: " $3}'
kubectl top pods -n kube-system -l component=etcd --no-headers 2>/dev/null | awk '{print "  etcd: " $3}'

echo ""
echo "Master Node Total Memory:"
kubectl top nodes -l node-role.kubernetes.io/control-plane --no-headers | awk '{print "  " $1 ": " $4 " / " $5}'
```

**Prometheus 알림 규칙:**

```yaml
groups:
- name: control_plane_memory
  rules:
  # Control Plane 메모리 사용량 알림
  - alert: HighControllerManagerMemory
    expr: container_memory_usage_bytes{pod=~"kube-controller-manager.*", namespace="kube-system"} > 1073741824  # 1GB
    for: 5m
    annotations:
      summary: "Controller Manager memory usage is high"
      description: "Controller Manager is using {{ $value | humanize1024 }} of memory"
  
  - alert: HighSchedulerMemory
    expr: container_memory_usage_bytes{pod=~"kube-scheduler.*", namespace="kube-system"} > 536870912  # 512MB
    for: 5m
    annotations:
      summary: "Scheduler memory usage is high"
      description: "Scheduler is using {{ $value | humanize1024 }} of memory"
  
  - alert: HighAPIServerMemory
    expr: container_memory_usage_bytes{pod=~"kube-apiserver.*", namespace="kube-system"} > 2147483648  # 2GB
    for: 5m
    annotations:
      summary: "API Server memory usage is high"
      description: "API Server is using {{ $value | humanize1024 }} of memory"
  
  - alert: HighEtcdMemory
    expr: container_memory_usage_bytes{pod=~"etcd.*", namespace="kube-system"} > 4294967296  # 4GB
    for: 5m
    annotations:
      summary: "etcd memory usage is high"
      description: "etcd is using {{ $value | humanize1024 }} of memory"
  
  # 마스터 노드 전체 메모리 알림
  - alert: HighMasterNodeMemory
    expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 80
    for: 5m
    annotations:
      summary: "Master node memory usage is high"
      description: "Master node {{ $labels.instance }} is using {{ $value }}% of memory"
```

### 7.3 Grafana 대시보드

**Control Plane 메모리 대시보드:**

```yaml
# Grafana Dashboard JSON
{
  "dashboard": {
    "title": "Control Plane Memory Usage",
    "panels": [
      {
        "title": "Controller Manager Memory",
        "targets": [
          {
            "expr": "container_memory_usage_bytes{pod=~'kube-controller-manager.*'}"
          }
        ]
      },
      {
        "title": "Scheduler Memory",
        "targets": [
          {
            "expr": "container_memory_usage_bytes{pod=~'kube-scheduler.*'}"
          }
        ]
      },
      {
        "title": "API Server Memory",
        "targets": [
          {
            "expr": "container_memory_usage_bytes{pod=~'kube-apiserver.*'}"
          }
        ]
      },
      {
        "title": "etcd Memory",
        "targets": [
          {
            "expr": "container_memory_usage_bytes{pod=~'etcd.*'}"
          }
        ]
      },
      {
        "title": "Pod Count vs Control Plane Memory",
        "targets": [
          {
            "expr": "count(kube_pod_info)",
            "legendFormat": "Pod Count"
          },
          {
            "expr": "sum(container_memory_usage_bytes{pod=~'kube-(controller-manager|scheduler|apiserver|etcd).*'})",
            "legendFormat": "Control Plane Memory"
          }
        ]
      }
    ]
  }
}
```

---

## 8. Best Practices

### 8.1 Control Plane 리소스 제한 설정 (필수)

**모든 Control Plane 컴포넌트에 리소스 제한 설정:**

```yaml
# kube-controller-manager
resources:
  requests:
    memory: "200Mi"
    cpu: "100m"
  limits:
    memory: "1Gi"
    cpu: "500m"

# kube-scheduler
resources:
  requests:
    memory: "100Mi"
    cpu: "100m"
  limits:
    memory: "500Mi"
    cpu: "500m"

# kube-apiserver
resources:
  requests:
    memory: "500Mi"
    cpu: "500m"
  limits:
    memory: "2Gi"
    cpu: "2000m"

# etcd
resources:
  requests:
    memory: "1Gi"
    cpu: "500m"
  limits:
    memory: "4Gi"
    cpu: "2000m"
```

### 8.2 etcd 최적화 설정

**etcd 압축 및 크기 제한:**

```yaml
# etcd.yaml
command:
- etcd
- --auto-compaction-mode=periodic
- --auto-compaction-retention=1h  # 1시간마다 압축
- --quota-backend-bytes=8589934592  # 8GB 제한
```

### 8.3 ResourceQuota로 Pod 수 제한

**네임스페이스별 Pod 수 제한:**

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: pod-limit
  namespace: default
spec:
  hard:
    pods: "100"  # 최대 100개 Pod
    requests.memory: "50Gi"
    limits.memory: "100Gi"
```

### 8.4 모니터링 및 알림 설정

**Control Plane 메모리 모니터링:**

- Prometheus로 Control Plane 컴포넌트 메모리 수집
- Grafana 대시보드로 시각화
- 알림 규칙으로 메모리 사용량 임계값 설정

---

## 9. 트러블슈팅

### 9.1 Control Plane 메모리 사용량이 계속 증가하는 경우

**확인 사항:**

1. **Control Plane 컴포넌트 메모리 확인:**
```bash
kubectl top pods -n kube-system
kubectl top pods -n kube-system -l component=kube-controller-manager
kubectl top pods -n kube-system -l component=kube-scheduler
kubectl top pods -n kube-system -l component=kube-apiserver
kubectl top pods -n kube-system -l component=etcd
```

2. **Pod 수 확인:**
```bash
kubectl get pods --all-namespaces | wc -l
```

3. **리소스 제한 확인:**
```bash
kubectl describe pod -n kube-system kube-controller-manager-<node> | grep -A 5 "Limits"
```

4. **etcd 크기 확인:**
```bash
kubectl exec -it -n kube-system etcd-<node> -- etcdctl endpoint status
```

### 9.2 해결 방법

**즉시 해결:**

```bash
# 1. 불필요한 Pod 정리
kubectl get pods --all-namespaces | grep -i pending
kubectl delete pod <pending-pod> --force --grace-period=0

# 2. etcd 수동 압축 (주의: 클러스터 중단 가능)
kubectl exec -it -n kube-system etcd-<node> -- etcdctl compact <revision>
kubectl exec -it -n kube-system etcd-<node> -- etcdctl defrag

# 3. Control Plane 컴포넌트 재시작 (주의: 일시적 중단)
kubectl delete pod -n kube-system kube-controller-manager-<node>
kubectl delete pod -n kube-system kube-scheduler-<node>
```

**근본 해결:**

```bash
# 1. Control Plane 리소스 제한 설정
sudo vi /etc/kubernetes/manifests/kube-controller-manager.yaml
# resources 섹션 추가

# 2. etcd 압축 설정
sudo vi /etc/kubernetes/manifests/etcd.yaml
# --auto-compaction-mode=periodic 추가

# 3. ResourceQuota 설정
kubectl apply -f resource-quota.yaml
```

### 9.3 메모리 부족 시 임시 조치

**Pod 수 감소:**

```bash
# 불필요한 Deployment 스케일 다운
kubectl scale deployment <deployment-name> --replicas=0

# 또는 특정 네임스페이스의 Pod 정리
kubectl delete pods --all -n <namespace>
```

**Control Plane 컴포넌트 리소스 증가 (임시):**

```yaml
# kube-controller-manager.yaml
resources:
  limits:
    memory: "2Gi"  # 임시로 증가
```

---

## 마무리

**핵심 포인트:**

- **Pod는 워커 노드에 배포되지만, Control Plane이 모든 Pod를 관리하여 마스터 노드 메모리를 사용합니다.**
- **Pod 수가 증가하면 Control Plane 컴포넌트의 메모리 사용량도 선형적으로 증가합니다.**
- **Control Plane 컴포넌트에 리소스 제한을 설정하여 메모리 사용량을 제어해야 합니다.**
- **etcd 압축 설정과 ResourceQuota를 통해 리소스 사용량을 관리해야 합니다.**
- **모니터링과 알림을 통해 Control Plane 메모리 사용량을 지속적으로 추적해야 합니다.**

Kubernetes 클러스터에서 **마스터 노드의 메모리는 Control Plane의 안정성**을 위해 신중하게 관리해야 합니다. Pod는 워커 노드에 배포되더라도, Control Plane이 모든 Pod를 관리하기 때문에 마스터 노드의 메모리 사용량이 증가할 수 있습니다. 적절한 리소스 제한과 모니터링을 통해 클러스터의 안정성을 보장해야 합니다.

다음 글에서는 Kubernetes의 **Resource Quota와 LimitRange**를 통해 리소스를 더 세밀하게 제어하는 방법을 정리해볼 예정입니다. 🚀

