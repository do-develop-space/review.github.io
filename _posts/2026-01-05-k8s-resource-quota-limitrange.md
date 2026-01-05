---
layout: post
title: "Kubernetes Resource Quota와 LimitRange: 리소스 제어 완전 가이드"
date: 2026-01-05
categories: [kubernetes, devops]
tags: [Kubernetes, ResourceQuota, LimitRange, 리소스제한, 메모리관리, CPU관리, 네임스페이스]
---

# Kubernetes Resource Quota와 LimitRange: 리소스 제어 완전 가이드

이전 글에서 Control Plane의 메모리 사용량을 제어하는 방법을 다뤘습니다. 이번 글에서는 **Resource Quota와 LimitRange**를 통해 네임스페이스와 Pod 레벨에서 리소스를 세밀하게 제어하는 방법을 정리해보겠습니다.

Kubernetes 클러스터에서 여러 팀이나 애플리케이션이 공유할 때, 리소스 사용량을 제한하고 관리하는 것은 매우 중요합니다. Resource Quota와 LimitRange를 통해 이를 효과적으로 관리할 수 있습니다.

---

## 1. Resource Quota란?

### 1.1 Resource Quota의 개념

**Resource Quota**는 네임스페이스 레벨에서 리소스 사용량을 제한하는 Kubernetes 리소스입니다.

**주요 기능:**
- 네임스페이스당 최대 리소스 사용량 제한
- Pod 수, CPU, 메모리, PVC(영구 볼륨 클레임) 등 제한
- 팀별/프로젝트별 리소스 격리

**사용 시나리오:**
- 여러 팀이 클러스터를 공유할 때
- 특정 애플리케이션이 과도한 리소스를 사용하지 않도록 제한
- 비용 관리 및 리소스 예산 관리

### 1.2 Resource Quota 예시

**기본 Resource Quota:**

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: production
spec:
  hard:
    # Pod 수 제한
    pods: "10"
    
    # CPU 제한
    requests.cpu: "4"
    limits.cpu: "8"
    
    # 메모리 제한
    requests.memory: "8Gi"
    limits.memory: "16Gi"
    
    # 영구 볼륨 클레임 제한
    persistentvolumeclaims: "4"
    requests.storage: "100Gi"
```

**적용 확인:**

```bash
# Resource Quota 생성
kubectl apply -f resource-quota.yaml

# Resource Quota 확인
kubectl get resourcequota -n production

# 상세 정보 확인
kubectl describe resourcequota compute-quota -n production
```

---

## 2. LimitRange란?

### 2.1 LimitRange의 개념

**LimitRange**는 네임스페이스 내의 개별 Pod나 Container에 대한 리소스 제한의 기본값과 최대/최소값을 설정하는 Kubernetes 리소스입니다.

**주요 기능:**
- Pod/Container의 기본 리소스 요청량/제한량 설정
- 최소/최대 리소스 제한 설정
- 리소스가 지정되지 않은 Pod에 자동으로 기본값 적용

**Resource Quota vs LimitRange:**

| 구분 | Resource Quota | LimitRange |
|------|---------------|------------|
| **범위** | 네임스페이스 전체 | 개별 Pod/Container |
| **목적** | 총 리소스 사용량 제한 | 개별 리소스 제한 |
| **기본값** | 제공 안 함 | 기본값 제공 가능 |

### 2.2 LimitRange 예시

**기본 LimitRange:**

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: mem-limit-range
  namespace: production
spec:
  limits:
  - default:
      memory: "512Mi"
      cpu: "500m"
    defaultRequest:
      memory: "256Mi"
      cpu: "250m"
    max:
      memory: "1Gi"
      cpu: "1000m"
    min:
      memory: "128Mi"
      cpu: "100m"
    type: Container
```

**적용 확인:**

```bash
# LimitRange 생성
kubectl apply -f limit-range.yaml

# LimitRange 확인
kubectl get limitrange -n production

# 상세 정보 확인
kubectl describe limitrange mem-limit-range -n production
```

---

## 3. Resource Quota 상세 가이드

### 3.1 리소스 타입별 제한

**CPU 및 메모리 제한:**

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: production
spec:
  hard:
    # 요청량(requests) 제한
    requests.cpu: "4"        # 총 CPU 요청량 4코어
    requests.memory: "8Gi"   # 총 메모리 요청량 8GB
    
    # 제한량(limits) 제한
    limits.cpu: "8"          # 총 CPU 제한량 8코어
    limits.memory: "16Gi"     # 총 메모리 제한량 16GB
```

**Pod 수 제한:**

```yaml
spec:
  hard:
    pods: "20"  # 최대 20개 Pod
```

**스토리지 제한:**

```yaml
spec:
  hard:
    persistentvolumeclaims: "10"  # 최대 10개 PVC
    requests.storage: "500Gi"     # 총 스토리지 요청량 500GB
    # 특정 스토리지 클래스별 제한
    requests.storage.storageclass.storage.k8s.io/requests.storage: "200Gi"
```

**서비스 및 로드밸런서 제한:**

```yaml
spec:
  hard:
    services: "10"                    # 최대 10개 Service
    services.loadbalancers: "2"        # 최대 2개 LoadBalancer
    services.nodeports: "5"            # 최대 5개 NodePort
```

**시크릿 및 ConfigMap 제한:**

```yaml
spec:
  hard:
    secrets: "20"      # 최대 20개 Secret
    configmaps: "20"   # 최대 20개 ConfigMap
```

### 3.2 리소스 사용량 확인

**현재 사용량 확인:**

```bash
# Resource Quota 사용량 확인
kubectl describe resourcequota compute-quota -n production

# 출력 예시:
# Name:            compute-quota
# Namespace:       production
# Resource         Used  Hard
# --------         ----  ----
# limits.cpu       6     8
# limits.memory    12Gi  16Gi
# pods              8    20
# requests.cpu      3    4
# requests.memory  6Gi  8Gi
```

**프로그래밍 방식으로 확인:**

```bash
# JSON 형식으로 확인
kubectl get resourcequota compute-quota -n production -o json

# 사용량만 추출
kubectl get resourcequota compute-quota -n production -o jsonpath='{.status.used}'
```

### 3.3 Resource Quota 위반 시 동작

**Pod 생성 시도 시:**

```yaml
# Resource Quota를 초과하는 Pod 생성 시도
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
  namespace: production
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        memory: "10Gi"  # Quota 초과 (8Gi 제한)
        cpu: "2"
```

**결과:**
- Pod 생성 실패
- 에러 메시지: "exceeded quota: compute-quota, requested: requests.memory=10Gi, used: 0, limited: requests.memory=8Gi"

---

## 4. LimitRange 상세 가이드

### 4.1 Container 리소스 제한

**기본값 및 제한 설정:**

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: container-limits
  namespace: production
spec:
  limits:
  - type: Container
    default:
      memory: "512Mi"
      cpu: "500m"
    defaultRequest:
      memory: "256Mi"
      cpu: "250m"
    max:
      memory: "2Gi"
      cpu: "2000m"
    min:
      memory: "128Mi"
      cpu: "100m"
```

**동작 방식:**

```yaml
# 리소스가 지정되지 않은 Pod
apiVersion: v1
kind: Pod
metadata:
  name: pod-without-resources
spec:
  containers:
  - name: app
    image: nginx
    # 리소스 미지정 → LimitRange의 default 적용
    # requests: memory=256Mi, cpu=250m
    # limits: memory=512Mi, cpu=500m
```

### 4.2 Pod 리소스 제한

**Pod 전체 리소스 제한:**

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: pod-limits
  namespace: production
spec:
  limits:
  - type: Pod
    max:
      memory: "4Gi"    # Pod 전체 최대 메모리
      cpu: "4000m"     # Pod 전체 최대 CPU
    min:
      memory: "256Mi"  # Pod 전체 최소 메모리
      cpu: "500m"      # Pod 전체 최소 CPU
```

**사용 예시:**

```yaml
# Pod에 여러 Container가 있는 경우
apiVersion: v1
kind: Pod
metadata:
  name: multi-container-pod
spec:
  containers:
  - name: app1
    image: nginx
    resources:
      requests:
        memory: "1Gi"
        cpu: "1000m"
  - name: app2
    image: nginx
    resources:
      requests:
        memory: "1Gi"
        cpu: "1000m"
  # Pod 전체: memory=2Gi, cpu=2000m (Pod LimitRange 내)
```

### 4.3 PVC(PersistentVolumeClaim) 리소스 제한

**PVC 크기 제한:**

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: pvc-limits
  namespace: production
spec:
  limits:
  - type: PersistentVolumeClaim
    max:
      storage: "100Gi"   # 최대 PVC 크기
    min:
      storage: "1Gi"     # 최소 PVC 크기
```

**사용 예시:**

```yaml
# PVC 생성 시
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
  namespace: production
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: "50Gi"  # LimitRange 범위 내 (1Gi ~ 100Gi)
```

### 4.4 LimitRange 위반 시 동작

**최대값 초과 시:**

```yaml
# LimitRange: max memory=2Gi
apiVersion: v1
kind: Pod
metadata:
  name: oversized-pod
spec:
  containers:
  - name: app
    image: nginx
    resources:
      limits:
        memory: "4Gi"  # LimitRange 최대값(2Gi) 초과
```

**결과:**
- Pod 생성 실패
- 에러 메시지: "memory limit 4Gi is greater than maximum allowed 2Gi"

**최소값 미만 시:**

```yaml
# LimitRange: min memory=128Mi
apiVersion: v1
kind: Pod
metadata:
  name: undersized-pod
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        memory: "64Mi"  # LimitRange 최소값(128Mi) 미만
```

**결과:**
- Pod 생성 실패
- 에러 메시지: "memory request 64Mi is less than minimum allowed 128Mi"

---

## 5. 실전 활용 예시

### 5.1 팀별 리소스 격리

**프로덕션 환경 설정:**

```yaml
# production 네임스페이스
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    pods: "50"
    requests.cpu: "20"
    requests.memory: "40Gi"
    limits.cpu: "40"
    limits.memory: "80Gi"
    persistentvolumeclaims: "20"
    requests.storage: "1Ti"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: production-limits
  namespace: production
spec:
  limits:
  - type: Container
    default:
      memory: "1Gi"
      cpu: "1000m"
    defaultRequest:
      memory: "512Mi"
      cpu: "500m"
    max:
      memory: "4Gi"
      cpu: "4000m"
    min:
      memory: "256Mi"
      cpu: "250m"
```

**개발 환경 설정:**

```yaml
# development 네임스페이스
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: development
spec:
  hard:
    pods: "20"
    requests.cpu: "4"
    requests.memory: "8Gi"
    limits.cpu: "8"
    limits.memory: "16Gi"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: dev-limits
  namespace: development
spec:
  limits:
  - type: Container
    default:
      memory: "256Mi"
      cpu: "250m"
    defaultRequest:
      memory: "128Mi"
      cpu: "100m"
    max:
      memory: "1Gi"
      cpu: "1000m"
    min:
      memory: "64Mi"
      cpu: "50m"
```

### 5.2 애플리케이션별 리소스 제한

**마이크로서비스별 설정:**

```yaml
# order-service 네임스페이스
apiVersion: v1
kind: ResourceQuota
metadata:
  name: order-service-quota
  namespace: order-service
spec:
  hard:
    pods: "10"
    requests.cpu: "4"
    requests.memory: "8Gi"
    limits.cpu: "8"
    limits.memory: "16Gi"
---
# payment-service 네임스페이스
apiVersion: v1
kind: ResourceQuota
metadata:
  name: payment-service-quota
  namespace: payment-service
spec:
  hard:
    pods: "5"
    requests.cpu: "2"
    requests.memory: "4Gi"
    limits.cpu: "4"
    limits.memory: "8Gi"
```

### 5.3 리소스 제한이 적용된 Deployment

**리소스가 지정된 Deployment:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: production
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: app
        image: my-app:latest
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
```

**리소스가 지정되지 않은 Deployment (LimitRange 기본값 적용):**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: production
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: app
        image: my-app:latest
        # resources 미지정
        # → LimitRange의 default 적용
        # requests: memory=512Mi, cpu=500m
        # limits: memory=1Gi, cpu=1000m
```

---

## 6. Resource Quota와 LimitRange 조합

### 6.1 함께 사용하는 이유

**Resource Quota:**
- 네임스페이스 전체 리소스 제한
- 팀별/프로젝트별 예산 관리

**LimitRange:**
- 개별 Pod/Container 리소스 제한
- 리소스 미지정 시 기본값 제공
- 최소/최대값으로 리소스 범위 제한

**조합 사용 예시:**

```yaml
# 1. Resource Quota: 네임스페이스 전체 제한
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
  namespace: team-a
spec:
  hard:
    pods: "20"
    requests.cpu: "10"
    requests.memory: "20Gi"
    limits.cpu: "20"
    limits.memory: "40Gi"

---
# 2. LimitRange: 개별 Pod 제한
apiVersion: v1
kind: LimitRange
metadata:
  name: team-limits
  namespace: team-a
spec:
  limits:
  - type: Container
    default:
      memory: "512Mi"
      cpu: "500m"
    defaultRequest:
      memory: "256Mi"
      cpu: "250m"
    max:
      memory: "2Gi"
      cpu: "2000m"
    min:
      memory: "128Mi"
      cpu: "100m"
```

**동작 방식:**

```
네임스페이스: team-a
├── Resource Quota: 총 20개 Pod, 총 10 CPU, 총 20Gi 메모리
└── LimitRange: 각 Pod 최대 2Gi 메모리, 최대 2 CPU

결과:
- 최대 20개 Pod 생성 가능
- 각 Pod는 최대 2Gi 메모리 사용 가능
- 총 메모리는 20Gi를 초과할 수 없음
```

### 6.2 실전 예시: 팀별 리소스 관리

**팀 A 설정:**

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-a-quota
  namespace: team-a
spec:
  hard:
    pods: "30"
    requests.cpu: "15"
    requests.memory: "30Gi"
    limits.cpu: "30"
    limits.memory: "60Gi"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: team-a-limits
  namespace: team-a
spec:
  limits:
  - type: Container
    default:
      memory: "1Gi"
      cpu: "1000m"
    max:
      memory: "4Gi"
      cpu: "4000m"
```

**팀 B 설정:**

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-b-quota
  namespace: team-b
spec:
  hard:
    pods: "15"
    requests.cpu: "8"
    requests.memory: "16Gi"
    limits.cpu: "16"
    limits.memory: "32Gi"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: team-b-limits
  namespace: team-b
spec:
  limits:
  - type: Container
    default:
      memory: "512Mi"
      cpu: "500m"
    max:
      memory: "2Gi"
      cpu: "2000m"
```

---

## 7. 모니터링 및 알림

### 7.1 Resource Quota 사용량 모니터링

**Prometheus 메트릭:**

```yaml
# kube-state-metrics를 통해 Resource Quota 메트릭 수집
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
# Resource Quota 사용률 대시보드
panels:
- title: "Resource Quota Usage"
  targets:
  - expr: |
      kube_resourcequota{resource="requests.cpu", namespace="production"}
      / 
      kube_resourcequota_hard{resource="requests.cpu", namespace="production"}
      * 100
    legendFormat: "CPU Usage %"
  - expr: |
      kube_resourcequota{resource="requests.memory", namespace="production"}
      / 
      kube_resourcequota_hard{resource="requests.memory", namespace="production"}
      * 100
    legendFormat: "Memory Usage %"
```

### 7.2 알림 규칙

**Resource Quota 사용률 알림:**

```yaml
groups:
- name: resource_quota
  rules:
  - alert: HighResourceQuotaUsage
    expr: |
      (kube_resourcequota{resource="requests.cpu"}
       / 
       kube_resourcequota_hard{resource="requests.cpu"}) * 100 > 80
    for: 5m
    annotations:
      summary: "Resource Quota usage is high"
      description: "Namespace {{ $labels.namespace }} is using {{ $value }}% of CPU quota"
  
  - alert: ResourceQuotaExceeded
    expr: |
      kube_resourcequota{resource="requests.cpu"}
      >= 
      kube_resourcequota_hard{resource="requests.cpu"}
    for: 1m
    annotations:
      summary: "Resource Quota exceeded"
      description: "Namespace {{ $labels.namespace }} has exceeded CPU quota"
```

---

## 8. Best Practices

### 8.1 Resource Quota 설정 권장사항

**1. 단계적 설정:**
- 초기에는 넉넉하게 설정
- 실제 사용량 모니터링 후 조정
- 점진적으로 최적화

**2. 팀별/환경별 분리:**
- 프로덕션/스테이징/개발 환경별로 다른 Quota 설정
- 팀별로 네임스페이스 분리 및 Quota 설정

**3. 여유 공간 확보:**
- 최대 사용량의 120% 정도로 설정
- 급작스러운 트래픽 증가 대비

### 8.2 LimitRange 설정 권장사항

**1. 기본값 설정:**
- 리소스 미지정 Pod에 자동 적용
- 개발자가 리소스를 지정하지 않아도 안전한 기본값 제공

**2. 최소/최대값 설정:**
- 최소값: 너무 작은 리소스로 인한 성능 문제 방지
- 최대값: 단일 Pod가 과도한 리소스를 사용하는 것 방지

**3. 환경별 차별화:**
- 프로덕션: 높은 기본값, 높은 최대값
- 개발: 낮은 기본값, 낮은 최대값

### 8.3 리소스 관리 체크리스트

**클러스터 초기 설정:**
- [ ] 네임스페이스별 Resource Quota 설정
- [ ] 네임스페이스별 LimitRange 설정
- [ ] 모니터링 및 알림 설정

**정기 점검:**
- [ ] Resource Quota 사용률 확인
- [ ] LimitRange 위반 사례 확인
- [ ] 리소스 사용 패턴 분석
- [ ] Quota 조정 필요성 검토

---

## 마무리

**핵심 포인트:**

- **Resource Quota는 네임스페이스 레벨에서 총 리소스 사용량을 제한합니다.**
- **LimitRange는 개별 Pod/Container의 리소스 제한과 기본값을 제공합니다.**
- **두 가지를 함께 사용하면 리소스를 효과적으로 관리할 수 있습니다.**
- **모니터링과 알림을 통해 리소스 사용량을 지속적으로 추적해야 합니다.**

Resource Quota와 LimitRange를 적절히 설정하면, 여러 팀이나 애플리케이션이 클러스터를 공유할 때 리소스 사용량을 효과적으로 제어하고 관리할 수 있습니다. 특히 Control Plane의 메모리 사용량을 제어하기 위해 Pod 수를 제한하는 Resource Quota는 매우 유용합니다.

다음 글에서는 **분산 환경에서 동시성 제어를 위한 분산 락(Distributed Lock)**에 대해 정리해볼 예정입니다. 🚀


