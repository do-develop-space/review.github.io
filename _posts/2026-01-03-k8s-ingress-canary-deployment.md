---
layout: post
title: "Kubernetes Ingress Canary 배포: 점진적 롤아웃 전략"
date: 2026-01-03
categories: [kubernetes, devops]
tags: [Kubernetes, Ingress, Canary, 점진적배포, 롤아웃, NGINX, 트래픽분산, A/B테스트]
---

# Kubernetes Ingress Canary 배포: 점진적 롤아웃 전략

이전 글에서 Kubernetes Ingress의 기본 개념과 라우팅 기능을 다뤘습니다. 이번 글에서는 **Canary 배포(Canary Deployment)**를 통해 새 버전을 안전하게 점진적으로 롤아웃하는 방법을 정리해보겠습니다.

Canary 배포는 새 버전의 애플리케이션을 일부 트래픽만 받도록 하여, 문제가 발생하지 않으면 점진적으로 트래픽을 늘려가는 배포 전략입니다. Kubernetes Ingress를 활용하면 코드 변경 없이 트래픽 분산을 제어할 수 있습니다.

---

## 1. Canary 배포란?

### 1.1 Canary 배포의 개념

**Canary 배포(Canary Deployment)**는 새 버전의 애플리케이션을 **일부 트래픽만 받도록** 하여, 문제가 없는지 확인한 후 점진적으로 트래픽을 늘려가는 배포 전략입니다.

**이름의 유래:**
- 과거 광부들이 새 광산에 들어갈 때 카나리아(Canary)를 먼저 보냄
- 카나리아가 죽으면 위험한 가스가 있다는 신호
- → 새 버전을 "카나리아"처럼 먼저 테스트

### 1.2 Canary 배포의 장점

**안전한 배포:**
- ✅ 새 버전의 문제를 조기에 발견
- ✅ 전체 사용자에게 영향을 주지 않음
- ✅ 빠른 롤백 가능

**점진적 검증:**
- ✅ 실제 프로덕션 트래픽으로 테스트
- ✅ 성능, 에러율, 사용자 반응 모니터링
- ✅ 문제 발견 시 즉시 롤백

**리스크 최소화:**
- ✅ 전체 트래픽을 한 번에 전환하지 않음
- ✅ 단계적으로 트래픽 증가
- ✅ 안정성 확인 후 전체 전환

### 1.3 Canary 배포 vs Blue-Green 배포

| 구분 | Canary 배포 | Blue-Green 배포 |
|------|------------|----------------|
| **트래픽 분산** | 점진적 (10% → 50% → 100%) | 즉시 전환 (0% → 100%) |
| **리소스 사용** | 적음 (기존 + 일부 새 버전) | 많음 (기존 + 전체 새 버전) |
| **롤백 속도** | 빠름 (트래픽만 조정) | 빠름 (전체 전환) |
| **리스크** | 낮음 (점진적) | 중간 (즉시 전환) |
| **사용 사례** | 안정적인 점진적 배포 | 빠른 전환이 필요한 경우 |

---

## 2. NGINX Ingress Controller의 Canary 기능

### 2.1 Canary 어노테이션

NGINX Ingress Controller는 **어노테이션(Annotation)**을 통해 Canary 배포를 지원합니다.

**주요 어노테이션:**

```yaml
annotations:
  # Canary 활성화
  nginx.ingress.kubernetes.io/canary: "true"
  
  # 트래픽 분산 비율 (0-100)
  nginx.ingress.kubernetes.io/canary-weight: "10"
  
  # 헤더 기반 라우팅
  nginx.ingress.kubernetes.io/canary-by-header: "X-Canary"
  nginx.ingress.kubernetes.io/canary-by-header-value: "true"
  
  # 쿠키 기반 라우팅
  nginx.ingress.kubernetes.io/canary-by-cookie: "canary"
```

### 2.2 Canary 트래픽 분산 방식

**1. Weight 기반 (가중치 기반)**
- 트래픽의 일정 비율을 새 버전으로 라우팅
- 예: 10% → 50% → 100%

**2. Header 기반**
- 특정 헤더가 있는 요청만 새 버전으로 라우팅
- 내부 테스트, 베타 사용자 등에 활용

**3. Cookie 기반**
- 특정 쿠키가 있는 요청만 새 버전으로 라우팅
- 사용자 세션 기반 라우팅

**우선순위:**
1. Header 기반 (가장 높음)
2. Cookie 기반
3. Weight 기반 (가장 낮음)

---

## 3. Weight 기반 Canary 배포

### 3.1 기본 구조

**기본 Ingress (Production):**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-production
  namespace: default
spec:
  ingressClassName: nginx
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-v1-service
            port:
              number: 80
```

**Canary Ingress (새 버전):**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-canary
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"  # 10% 트래픽
spec:
  ingressClassName: nginx
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-v2-service
            port:
              number: 80
```

### 3.2 점진적 롤아웃 예시

**Step 1: 10% 트래픽 분산**

```yaml
annotations:
  nginx.ingress.kubernetes.io/canary: "true"
  nginx.ingress.kubernetes.io/canary-weight: "10"
```

**Step 2: 50% 트래픽 분산**

```yaml
annotations:
  nginx.ingress.kubernetes.io/canary: "true"
  nginx.ingress.kubernetes.io/canary-weight: "50"
```

**Step 3: 100% 트래픽 전환 (Canary 비활성화)**

```yaml
# Canary Ingress 삭제 또는
annotations:
  nginx.ingress.kubernetes.io/canary: "false"
```

**Production Ingress 업데이트:**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-production
spec:
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-v2-service  # v2로 변경
            port:
              number: 80
```

### 3.3 실전 예제

**전체 배포 구조:**

```yaml
# 1. V1 Deployment (기존 버전)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-v1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: app
      version: v1
  template:
    metadata:
      labels:
        app: app
        version: v1
    spec:
      containers:
      - name: app
        image: myapp:v1
        ports:
        - containerPort: 8080
---
# 2. V1 Service
apiVersion: v1
kind: Service
metadata:
  name: app-v1-service
spec:
  selector:
    app: app
    version: v1
  ports:
  - port: 80
    targetPort: 8080
---
# 3. V2 Deployment (새 버전)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-v2
spec:
  replicas: 1  # 초기에는 적은 수의 Pod
  selector:
    matchLabels:
      app: app
      version: v2
  template:
    metadata:
      labels:
        app: app
        version: v2
    spec:
      containers:
      - name: app
        image: myapp:v2
        ports:
        - containerPort: 8080
---
# 4. V2 Service
apiVersion: v1
kind: Service
metadata:
  name: app-v2-service
spec:
  selector:
    app: app
    version: v2
  ports:
  - port: 80
    targetPort: 8080
---
# 5. Production Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-production
spec:
  ingressClassName: nginx
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-v1-service
            port:
              number: 80
---
# 6. Canary Ingress (10% 트래픽)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-canary
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"
spec:
  ingressClassName: nginx
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-v2-service
            port:
              number: 80
```

---

## 4. Header 기반 Canary 배포

### 4.1 내부 테스트용 Canary

**특정 헤더가 있는 요청만 새 버전으로 라우팅:**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-canary
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-by-header: "X-Canary"
    nginx.ingress.kubernetes.io/canary-by-header-value: "true"
spec:
  ingressClassName: nginx
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-v2-service
            port:
              number: 80
```

**사용 방법:**

```bash
# 일반 요청 (V1으로 라우팅)
curl http://app.example.com/api/users

# Canary 헤더 포함 요청 (V2로 라우팅)
curl -H "X-Canary: true" http://app.example.com/api/users
```

### 4.2 베타 사용자용 Canary

**특정 사용자 그룹만 새 버전으로 라우팅:**

```yaml
annotations:
  nginx.ingress.kubernetes.io/canary: "true"
  nginx.ingress.kubernetes.io/canary-by-header: "X-User-Type"
  nginx.ingress.kubernetes.io/canary-by-header-value: "beta"
```

**사용 예시:**

```bash
# 베타 사용자만 V2로 라우팅
curl -H "X-User-Type: beta" http://app.example.com/api/users
```

---

## 5. Cookie 기반 Canary 배포

### 5.1 쿠키 기반 라우팅

**특정 쿠키가 있는 요청만 새 버전으로 라우팅:**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-canary
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-by-cookie: "canary"
spec:
  ingressClassName: nginx
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-v2-service
            port:
              number: 80
```

**사용 방법:**

```bash
# 쿠키가 있는 요청만 V2로 라우팅
curl -b "canary=always" http://app.example.com/api/users
```

**브라우저에서 테스트:**

```javascript
// 쿠키 설정
document.cookie = "canary=always";

// 이후 모든 요청이 V2로 라우팅됨
```

---

## 6. 하이브리드 Canary 전략

### 6.1 Weight + Header 조합

**일반 사용자: Weight 기반 분산**
**내부 테스트: Header 기반 강제 라우팅**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-canary
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"  # 일반 사용자 10%
    nginx.ingress.kubernetes.io/canary-by-header: "X-Internal-Test"
    nginx.ingress.kubernetes.io/canary-by-header-value: "true"  # 내부 테스트 100%
spec:
  ingressClassName: nginx
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-v2-service
            port:
              number: 80
```

### 6.2 단계별 Canary 배포 전략

**Phase 1: 내부 테스트 (Header 기반)**
```yaml
annotations:
  nginx.ingress.kubernetes.io/canary: "true"
  nginx.ingress.kubernetes.io/canary-by-header: "X-Internal"
  nginx.ingress.kubernetes.io/canary-by-header-value: "true"
```

**Phase 2: 소규모 트래픽 (Weight 5%)**
```yaml
annotations:
  nginx.ingress.kubernetes.io/canary: "true"
  nginx.ingress.kubernetes.io/canary-weight: "5"
```

**Phase 3: 중간 규모 트래픽 (Weight 25%)**
```yaml
annotations:
  nginx.ingress.kubernetes.io/canary: "true"
  nginx.ingress.kubernetes.io/canary-weight: "25"
```

**Phase 4: 대규모 트래픽 (Weight 50%)**
```yaml
annotations:
  nginx.ingress.kubernetes.io/canary: "true"
  nginx.ingress.kubernetes.io/canary-weight: "50"
```

**Phase 5: 전체 전환 (Weight 100% → Production 전환)**

---

## 7. 모니터링 및 롤백

### 7.1 모니터링 지표

**핵심 메트릭:**

1. **에러율 (Error Rate)**
   - HTTP 5xx 에러 비율
   - V2의 에러율이 V1보다 높으면 롤백

2. **응답 시간 (Latency)**
   - P50, P95, P99 응답 시간
   - V2의 응답 시간이 크게 증가하면 롤백

3. **처리량 (Throughput)**
   - 초당 요청 수 (RPS)
   - V2의 처리량이 크게 감소하면 롤백

4. **비즈니스 메트릭**
   - 전환율, 매출 등
   - V2에서 비즈니스 지표가 악화되면 롤백

### 7.2 Prometheus + Grafana 모니터링

**NGINX Ingress 메트릭 수집:**

```yaml
# Prometheus ServiceMonitor
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: nginx-ingress
spec:
  selector:
    matchLabels:
      app: ingress-nginx
  endpoints:
  - port: metrics
    interval: 30s
```

**Grafana 대시보드 예시:**

```yaml
# 주요 쿼리
# 에러율
rate(nginx_ingress_controller_requests{status=~"5.."}[5m])

# 응답 시간
histogram_quantile(0.95, 
  rate(nginx_ingress_controller_response_duration_seconds_bucket[5m])
)

# 트래픽 분산
sum(rate(nginx_ingress_controller_requests{service="app-v1-service"}[5m]))
sum(rate(nginx_ingress_controller_requests{service="app-v2-service"}[5m]))
```

### 7.3 자동 롤백 전략

**에러율 기반 자동 롤백:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: app-rollout
spec:
  strategy:
    canary:
      steps:
      - setWeight: 10
      - setWeight: 25
      - setWeight: 50
      - setWeight: 100
      analysis:
        templates:
        - templateName: error-rate
        args:
        - name: service-name
          value: app-v2-service
        startingStep: 2
        interval: 5m
```

**수동 롤백:**

```bash
# Canary Ingress 삭제
kubectl delete ingress app-canary

# 또는 Weight를 0으로 설정
kubectl annotate ingress app-canary \
  nginx.ingress.kubernetes.io/canary-weight="0" --overwrite
```

---

## 8. 실전 Best Practices

### 8.1 Canary 배포 체크리스트

**배포 전:**
- [ ] 모니터링 대시보드 준비
- [ ] 롤백 계획 수립
- [ ] 알림 설정 (Slack, PagerDuty 등)
- [ ] 테스트 시나리오 준비

**배포 중:**
- [ ] 단계별 트래픽 증가 (10% → 25% → 50% → 100%)
- [ ] 각 단계마다 최소 15-30분 모니터링
- [ ] 에러율, 응답 시간, 비즈니스 메트릭 확인
- [ ] 문제 발견 시 즉시 롤백

**배포 후:**
- [ ] 전체 트래픽 전환 확인
- [ ] 모니터링 지속 (최소 1시간)
- [ ] Canary Ingress 정리

### 8.2 트래픽 분산 비율 권장사항

**보수적 접근:**
- 5% → 15% → 30% → 50% → 100%
- 각 단계: 30분 이상

**적극적 접근:**
- 10% → 50% → 100%
- 각 단계: 15분 이상

**빠른 전환:**
- 25% → 75% → 100%
- 각 단계: 10분 이상 (리스크 높음)

### 8.3 주의사항

**1. Pod 수 조정**
- Canary 초기에는 적은 수의 Pod로 시작
- 트래픽 증가에 따라 Pod 수 증가

```yaml
# 초기: 1개 Pod
replicas: 1

# 50% 트래픽: 3개 Pod
replicas: 3

# 100% 트래픽: 5개 Pod
replicas: 5
```

**2. 데이터베이스 마이그레이션**
- 스키마 변경이 있는 경우 주의
- 양방향 호환성 유지
- 롤백 시 데이터 일관성 확인

**3. 외부 API 의존성**
- 외부 API 변경 시 호환성 확인
- Feature Flag 활용

**4. 세션 관리**
- Sticky Session 사용 시 주의
- Canary와 Production 간 세션 공유 확인

---

## 9. 실전 예제: 완전한 Canary 배포 파이프라인

### 9.1 전체 구조

```yaml
# 1. V1 Deployment (기존)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-v1
spec:
  replicas: 5
  selector:
    matchLabels:
      app: app
      version: v1
  template:
    metadata:
      labels:
        app: app
        version: v1
    spec:
      containers:
      - name: app
        image: myapp:v1
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
---
# 2. V1 Service
apiVersion: v1
kind: Service
metadata:
  name: app-v1-service
spec:
  selector:
    app: app
    version: v1
  ports:
  - port: 80
    targetPort: 8080
---
# 3. V2 Deployment (새 버전)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-v2
spec:
  replicas: 1  # 초기에는 1개
  selector:
    matchLabels:
      app: app
      version: v2
  template:
    metadata:
      labels:
        app: app
        version: v2
    spec:
      containers:
      - name: app
        image: myapp:v2
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
---
# 4. V2 Service
apiVersion: v1
kind: Service
metadata:
  name: app-v2-service
spec:
  selector:
    app: app
    version: v2
  ports:
  - port: 80
    targetPort: 8080
---
# 5. Production Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-production
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - app.example.com
    secretName: app-tls
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-v1-service
            port:
              number: 80
---
# 6. Canary Ingress (10% 트래픽)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-canary
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - app.example.com
    secretName: app-tls
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-v2-service
            port:
              number: 80
```

### 9.2 단계별 배포 스크립트

```bash
#!/bin/bash

# Canary 배포 스크립트

NAMESPACE="default"
CANARY_INGRESS="app-canary"
PRODUCTION_INGRESS="app-production"
V2_SERVICE="app-v2-service"

# Step 1: 10% 트래픽
echo "Step 1: 10% 트래픽으로 Canary 배포 시작"
kubectl annotate ingress $CANARY_INGRESS \
  nginx.ingress.kubernetes.io/canary-weight="10" \
  -n $NAMESPACE --overwrite

echo "15분 대기 중..."
sleep 900

# 모니터링 확인 (에러율 체크)
ERROR_RATE=$(kubectl get --raw /apis/metrics.k8s.io/v1beta1/namespaces/$NAMESPACE/pods | \
  jq '.items[] | select(.metadata.name | startswith("app-v2")) | ...')

if [ "$ERROR_RATE" -gt "5" ]; then
  echo "에러율이 5%를 초과했습니다. 롤백합니다."
  kubectl delete ingress $CANARY_INGRESS -n $NAMESPACE
  exit 1
fi

# Step 2: 25% 트래픽
echo "Step 2: 25% 트래픽으로 증가"
kubectl annotate ingress $CANARY_INGRESS \
  nginx.ingress.kubernetes.io/canary-weight="25" \
  -n $NAMESPACE --overwrite

# V2 Pod 수 증가
kubectl scale deployment app-v2 --replicas=2 -n $NAMESPACE

echo "15분 대기 중..."
sleep 900

# Step 3: 50% 트래픽
echo "Step 3: 50% 트래픽으로 증가"
kubectl annotate ingress $CANARY_INGRESS \
  nginx.ingress.kubernetes.io/canary-weight="50" \
  -n $NAMESPACE --overwrite

# V2 Pod 수 증가
kubectl scale deployment app-v2 --replicas=3 -n $NAMESPACE

echo "15분 대기 중..."
sleep 900

# Step 4: 100% 트래픽 (Production 전환)
echo "Step 4: 100% 트래픽으로 전환"
kubectl annotate ingress $PRODUCTION_INGRESS \
  -n $NAMESPACE --overwrite \
  -o yaml | \
  sed "s/app-v1-service/app-v2-service/g" | \
  kubectl apply -f -

# V2 Pod 수 최종 증가
kubectl scale deployment app-v2 --replicas=5 -n $NAMESPACE

# Canary Ingress 삭제
kubectl delete ingress $CANARY_INGRESS -n $NAMESPACE

echo "Canary 배포 완료!"
```

---

## 10. 트러블슈팅

### 10.1 Canary가 동작하지 않는 경우

**문제: Canary Ingress가 트래픽을 받지 않음**

**원인:**
- 어노테이션 오타
- Host 이름 불일치
- Ingress Controller가 Canary를 인식하지 못함

**해결:**
```bash
# Ingress 확인
kubectl describe ingress app-canary

# 어노테이션 확인
kubectl get ingress app-canary -o yaml | grep canary

# NGINX Ingress Controller 로그 확인
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller
```

### 10.2 트래픽 분산 비율이 정확하지 않은 경우

**문제: Weight 10%로 설정했는데 실제로는 다른 비율**

**원인:**
- NGINX Ingress의 Weight는 정확하지 않을 수 있음
- 트래픽이 적을 때는 통계적 변동이 큼

**해결:**
- 충분한 트래픽 확보
- 모니터링으로 실제 비율 확인
- Weight 대신 Header/Cookie 기반 사용 고려

### 10.3 세션 문제

**문제: 사용자가 V1과 V2 사이를 왔다갔다 함**

**원인:**
- Sticky Session 미사용
- 쿠키 기반 라우팅 미사용

**해결:**
```yaml
# Sticky Session 활성화
annotations:
  nginx.ingress.kubernetes.io/affinity: "cookie"
  nginx.ingress.kubernetes.io/session-cookie-name: "route"
  nginx.ingress.kubernetes.io/session-cookie-expires: "172800"
  nginx.ingress.kubernetes.io/session-cookie-max-age: "172800"
```

---

## 마무리

**핵심 포인트:**

- **Canary 배포는 새 버전을 안전하게 점진적으로 롤아웃하는 핵심 전략입니다.**
- **NGINX Ingress Controller의 어노테이션을 통해 Weight, Header, Cookie 기반 라우팅이 가능합니다.**
- **단계별 트래픽 증가와 모니터링을 통해 문제를 조기에 발견하고 롤백할 수 있습니다.**
- **실제 프로덕션 트래픽으로 테스트하여 성능과 안정성을 검증할 수 있습니다.**

Kubernetes Ingress를 활용한 Canary 배포는 **코드 변경 없이 트래픽 분산을 제어**할 수 있어, 안전하고 효율적인 배포 전략입니다. 특히 마이크로서비스 아키텍처에서 각 서비스를 독립적으로 배포할 때 매우 유용합니다.

다음 글에서는 **워커 노드에 Pod를 배포했는데도 마스터 노드의 메모리 사용량이 증가하는 문제**와 Control Plane 관리 오버헤드에 대해 정리해볼 예정입니다. 🚀

