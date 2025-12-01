---
layout: post
title: "Kubernetes Control Plane과 Ingress 정리"
date: 2025-12-01
categories: [kubernetes]
tags: [Kubernetes, K8S, Control Plane, Ingress, 클러스터, 컨테이너오케스트레이션]
---

# Kubernetes Control Plane과 Ingress 정리

Kubernetes는 컨테이너화된 애플리케이션의 배포, 확장, 관리를 자동화하는 오픈소스 플랫폼입니다. Kubernetes 클러스터는 **Control Plane(컨트롤 플레인)**과 **Worker Node(워커 노드)**로 구성되며, 외부 트래픽을 관리하기 위해 **Ingress**를 사용합니다. 이번 포스트에서는 Control Plane의 역할과 Ingress의 개념을 자세히 알아보겠습니다.

## Kubernetes Control Plane이란?

**Control Plane**은 Kubernetes 클러스터의 두뇌 역할을 하는 핵심 컴포넌트들의 집합입니다. 클러스터의 상태를 관리하고, 사용자의 요청을 처리하며, 워커 노드와 통신하여 애플리케이션을 실행합니다.

### Control Plane의 주요 컴포넌트

#### 1. kube-apiserver (API Server)

**역할:**
- Kubernetes API의 프론트엔드 역할
- 모든 클러스터 작업의 진입점
- RESTful API를 통해 클러스터 상태를 조회하고 변경

**주요 기능:**
- 인증 및 인가 처리
- 요청 검증 및 어댑션(admission control)
- etcd와 통신하여 클러스터 상태 저장/조회

#### 2. etcd

**역할:**
- 클러스터의 모든 설정 데이터와 상태 정보를 저장하는 분산 키-값 저장소
- 클러스터의 "단일 진실 공급원(Single Source of Truth)"

**저장되는 정보:**
- Pod, Service, Deployment 등의 리소스 정보
- 클러스터 설정
- 노드 상태 정보

#### 3. kube-scheduler (Scheduler)

**역할:**
- 새로 생성된 Pod를 적절한 워커 노드에 할당하는 스케줄러

**스케줄링 고려사항:**
- 리소스 요구사항 (CPU, 메모리)
- 하드웨어/소프트웨어 제약사항
- 어피니티(affinity) 및 안티-어피니티(anti-affinity) 규칙
- 데이터 지역성

#### 4. kube-controller-manager (Controller Manager)

**역할:**
- 클러스터의 상태를 모니터링하고 조정하는 컨트롤러들을 실행

**주요 컨트롤러:**
- **Deployment Controller**: Deployment의 desired state 유지
- **ReplicaSet Controller**: Pod 복제본 수 관리
- **Node Controller**: 노드 상태 모니터링 및 관리
- **Service Controller**: Service와 LoadBalancer 관리
- **Namespace Controller**: Namespace 생명주기 관리

#### 5. cloud-controller-manager (Cloud Controller Manager)

**역할:**
- 클라우드 제공자별 로직을 실행하는 컨트롤러

**주요 기능:**
- 노드 라우팅 관리
- 볼륨 관리
- 로드 밸런서 관리
- 네트워크 관리

### Control Plane의 동작 흐름

```
사용자 요청 (kubectl apply)
    ↓
kube-apiserver (API 검증 및 처리)
    ↓
etcd (상태 저장)
    ↓
kube-controller-manager (상태 조정)
    ↓
kube-scheduler (Pod 스케줄링)
    ↓
워커 노드에 Pod 배포
```

## Ingress란?

**Ingress**는 클러스터 외부에서 내부 서비스로의 HTTP 및 HTTPS 트래픽을 관리하는 Kubernetes 리소스입니다. 하나의 로드 밸런서 IP를 사용하여 여러 서비스에 접근할 수 있게 해줍니다.

### Ingress의 필요성

**Service의 한계:**
- `NodePort`: 모든 노드의 포트를 열어야 함
- `LoadBalancer`: 서비스마다 별도의 로드 밸런서 필요 (비용 증가)
- `ClusterIP`: 클러스터 내부에서만 접근 가능

**Ingress의 장점:**
- 하나의 IP로 여러 서비스 라우팅
- 도메인 기반 라우팅
- SSL/TLS 종료
- 경로 기반 라우팅

### Ingress의 구성 요소

#### 1. Ingress Resource

**역할:**
- 트래픽 라우팅 규칙을 정의하는 Kubernetes 리소스

**주요 설정:**
- 호스트(host) 기반 라우팅
- 경로(path) 기반 라우팅
- 백엔드 서비스 지정
- TLS 인증서 설정

#### 2. Ingress Controller

**역할:**
- Ingress 리소스를 실제로 처리하고 트래픽을 라우팅하는 컴포넌트

**대표적인 Ingress Controller:**
- **NGINX Ingress Controller**: 가장 널리 사용됨
- **Traefik**: 자동 인증서 관리 지원
- **HAProxy Ingress**: 고성능이 필요한 경우
- **AWS ALB Ingress Controller**: AWS 환경에서 사용

### Ingress 리소스 예시

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - example.com
    secretName: example-tls
  rules:
  - host: example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
      - path: /web
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

### Ingress의 주요 기능

#### 1. 호스트 기반 라우팅

```yaml
rules:
- host: api.example.com
  http:
    paths:
    - backend:
        service:
          name: api-service
          port:
            number: 80
- host: web.example.com
  http:
    paths:
    - backend:
        service:
          name: web-service
          port:
            number: 80
```

#### 2. 경로 기반 라우팅

```yaml
rules:
- host: example.com
  http:
    paths:
    - path: /v1
      backend:
        service:
          name: v1-service
    - path: /v2
      backend:
        service:
          name: v2-service
```

#### 3. SSL/TLS 종료

```yaml
tls:
- hosts:
  - example.com
  secretName: example-tls-secret
```

#### 4. 리라이트(Rewrite) 규칙

```yaml
annotations:
  nginx.ingress.kubernetes.io/rewrite-target: /$1
```

## Control Plane과 Ingress의 관계

### 트래픽 흐름

```
외부 요청
    ↓
Ingress Controller (워커 노드에서 실행)
    ↓
Ingress Resource (Control Plane의 etcd에 저장)
    ↓
kube-apiserver (Ingress 리소스 조회)
    ↓
Ingress Controller가 라우팅 규칙 적용
    ↓
Service → Pod로 트래픽 전달
```

### Ingress Controller의 배포

**Ingress Controller는 일반적으로:**
- DaemonSet 또는 Deployment로 배포
- 워커 노드에서 실행되거나 별도의 노드에 배치
- LoadBalancer 또는 NodePort Service로 노출

## 모범 사례

### Control Plane 관리

1. **고가용성 구성**
   - Control Plane 컴포넌트를 여러 노드에 분산 배치
   - etcd 클러스터 구성 (최소 3개 노드)

2. **리소스 할당**
   - 충분한 CPU와 메모리 할당
   - etcd는 SSD 스토리지 사용 권장

3. **백업**
   - etcd 정기 백업
   - 클러스터 설정 백업

### Ingress 관리

1. **Ingress Controller 선택**
   - 환경에 맞는 Controller 선택
   - 성능과 기능 요구사항 고려

2. **TLS/SSL 인증서 관리**
   - Let's Encrypt와 같은 자동 인증서 관리 도구 활용
   - Secret으로 인증서 저장

3. **리소스 제한**
   - Ingress Controller에 적절한 리소스 제한 설정
   - Rate Limiting 설정

4. **모니터링**
   - Ingress 트래픽 모니터링
   - 에러 로그 추적

**향후 Kubernetes 환경으로 마이그레이션할 때, NGINX Ingress Controller를 사용하여 여러 마이크로서비스를 단일 진입점으로 관리할 계획입니다.** Control Plane의 고가용성 구성과 Ingress의 적절한 설정이 안정적인 서비스 운영의 핵심입니다.

---

다음 포스트에서는 **Minikube의 개념과 터널 기능**에 대해 다루겠습니다. 로컬 환경에서 Kubernetes 클러스터를 실행하고 테스트할 수 있는 Minikube의 사용법과, 터널을 통한 서비스 접근 방법을 정리해보겠습니다. 계속해서 K8s 관련 내용을 공유하겠습니다! 🚀


