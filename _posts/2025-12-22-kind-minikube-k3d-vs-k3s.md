---
layout: post
title: "kind / minikube / k3d vs k3s: 로컬 및 엣지 Kubernetes 환경 비교"
date: 2025-12-22
categories: [kubernetes]
tags: [Kubernetes, kind, minikube, k3d, k3s, 로컬클러스터, 엣지]
---

# kind / minikube / k3d vs k3s: 로컬 및 엣지 Kubernetes 환경 비교

쿠버네티스를 공부하거나 개발 환경에서 테스트할 때, 가장 먼저 고민하는 것이 **"로컬에서 쿠버네티스 클러스터를 어떻게 띄울까?"** 입니다.  
대표적인 선택지가 **kind, minikube, k3d**, 그리고 **k3s**인데, 이름도 비슷하고 역할도 겹쳐서 헷갈리기 쉽습니다.

이 글에서는 **kind / minikube / k3d**와 **k3s**의 특징과 차이점을 정리하고, 어떤 상황에서 무엇을 선택하면 좋을지 비교해보겠습니다.

---

## 1. 한 줄 정의로 정리하기

먼저 각 도구를 한 줄로 정리하면 다음과 같습니다:

- **kind**: Docker 컨테이너 위에서 **Kubernetes를 실행하는 도구** (테스트/CI용 로컬 클러스터)
- **minikube**: VM 또는 컨테이너 위에서 **싱글 노드(또는 소규모) Kubernetes 클러스터를 띄워주는 로컬 개발용 도구**
- **k3d**: **k3s를 Docker 컨테이너 위에서 실행**해주는 도구 (k3s + Docker = k3d)
- **k3s**: 경량화된 Kubernetes 배포판, **엣지/IoT/리소스 적은 환경**을 위한 경량 Kubernetes

정리하면:
- **kind / minikube / k3d**: "클러스터를 띄워주는 도구" (로컬/테스트용 런타임)
- **k3s**: "경량 Kubernetes 배포판" (도구라기보다 **Kubernetes의 한 종류**)

---

## 2. kind: Kubernetes IN Docker

### 개념

**kind(Kubernetes IN Docker)**는 이름 그대로 **Docker 컨테이너 안에서 Kubernetes 클러스터를 실행하는 도구**입니다.

- 공식 Kubernetes 프로젝트에서 제공 (`sig-testing`)
- 주로 **테스트/CI 환경**에 최적화
- 클러스터의 각 노드가 Docker 컨테이너로 실행됨

### 특징

- **장점:**
  - 설치/삭제가 매우 빠름 (`kind create cluster`, `kind delete cluster`)
  - CI 파이프라인에서 자동으로 클러스터 생성/삭제하기 좋음
  - Docker만 있으면 동작 (추가 VM 설치 불필요)
  - 공식 Kubernetes 테스트에 사용될 정도로 안정적

- **단점:**
  - 로컬 개발용으로 충분하지만, **실제 운영 환경과는 조금 다를 수 있음** (특히 네트워킹/LoadBalancer 부분)
  - Ingress/LoadBalancer 설정이 약간 번거로울 수 있음 (별도 설정 필요)

### 사용 예시

```bash
# 클러스터 생성
kind create cluster --name dev-cluster

# kubeconfig 자동 설정 (kubectl로 바로 사용 가능)
kubectl get nodes

# 클러스터 삭제
kind delete cluster --name dev-cluster
```

**추천 사용 사례:**
- CI에서 테스트용 Kubernetes 클러스터가 필요할 때
- 로컬에서 빠르게 여러 개의 Kubernetes 버전을 테스트할 때

---

## 3. minikube: 로컬 개발용 Kubernetes 올인원

### 개념

**minikube**는 로컬에서 Kubernetes 클러스터를 쉽게 띄울 수 있게 해주는 도구입니다.

- 예전부터 가장 널리 사용된 로컬 Kubernetes 도구
- **VM, Docker, containerd 등 다양한 드라이버 지원**

### 특징

- **장점:**
  - `minikube addons enable ingress` 등으로 **Ingress, Dashboard 등 애드온 지원**
  - `minikube tunnel`로 LoadBalancer 타입 서비스도 로컬에서 테스트 가능
  - 다양한 드라이버: Docker, HyperKit, VirtualBox 등

- **단점:**
  - kind에 비해 상대적으로 무겁고 느릴 수 있음
  - 설정 옵션이 많아서 처음에는 다소 복잡하게 느껴질 수 있음

### 사용 예시

```bash
# Docker 드라이버로 클러스터 생성
minikube start --driver=docker

# Ingress 애드온 활성화
minikube addons enable ingress

# 대시보드 실행
minikube dashboard

# LoadBalancer 서비스 접근 (필요 시)
minikube tunnel
```

**추천 사용 사례:**
- 로컬에서 **Ingress, LoadBalancer, Dashboard까지 포함한 풀스택 환경**을 구성해보고 싶을 때
- 쿠버네티스를 처음 접하는 학습용 환경

---

## 4. k3s: 경량 Kubernetes 배포판

### 개념

**k3s**는 Rancher에서 만든 **경량화된 Kubernetes 배포판**입니다.

- "K8s에서 불필요한 것들을 덜어낸 경량 버전"이라는 컨셉
- 단일 바이너리(약 100MB 내외)로 구성
- **엣지/IoT/리소스 제한 환경**에서 사용하기 좋음

### 특징

- **장점:**
  - 경량: 기본적으로 많은 기능이 통합/최적화되어 있음 (예: 내장 SQLite, 경량 컴포넌트)
  - 설치가 매우 간단: `curl | sh` 한 줄 설치 가능
  - ARM(라즈베리파이 등) 환경에서도 잘 동작

- **단점:**
  - 완전히 표준 Kubernetes이긴 하지만, **클라우드 매니지드 K8s(EKS/GKE/AKS)와는 구성 차이**가 있음
  - 로컬 개발보다는 실제 엣지/온프레미스 환경을 타깃으로 하는 경우가 많음

### 설치 예시 (단일 노드)

```bash
curl -sfL https://get.k3s.io | sh -

# kubeconfig
sudo cat /etc/rancher/k3s/k3s.yaml > ~/.kube/config
kubectl get nodes
```

**추천 사용 사례:**
- 라즈베리파이 클러스터, 엣지 디바이스, 온프레미스 소규모 환경
- 경량/소형 Kubernetes 클러스터가 필요한 경우

---

## 5. k3d: Docker 위에서 k3s 실행하기

### 개념

**k3d**는 **k3s를 Docker 컨테이너로 실행**해 주는 도구입니다.

- kind가 "Kubernetes in Docker"라면, k3d는 "k3s in Docker"에 해당
- k3s의 경량성과 Docker 기반 실행의 편리함을 결합

### 특징

- **장점:**
  - k3s를 로컬에서 빠르게 띄울 수 있음
  - 여러 노드(멀티 노드) 구성도 간단하게 가능
  - kind처럼 Docker만 있으면 동작

- **단점:**
  - 개념적으로 kind와 비슷해서, 둘 중 어떤 걸 쓸지 선택이 필요
  - pure k3s(실제 서버 설치)와는 약간의 차이가 있음

### 사용 예시

```bash
# k3d 설치 (예: Mac)
brew install k3d

# 단일 클러스터 생성
k3d cluster create dev-cluster

# kubeconfig 자동 설정 후 사용
kubectl get nodes

# 클러스터 삭제
k3d cluster delete dev-cluster
```

**추천 사용 사례:**
- k3s 기반 환경을 로컬에서 재현하고 싶을 때
- kind 대신 k3s 생태계를 그대로 활용하고 싶을 때

---

## 6. kind / minikube / k3d vs k3s 비교 정리

### 역할 관점 비교

| 도구 | 역할 | 실행 환경 | 주요 용도 |
|------|------|-----------|-----------|
| **kind** | Docker 위에 Kubernetes 실행 도구 | Docker 컨테이너 | 테스트/CI, 로컬 개발 |
| **minikube** | 로컬 Kubernetes 올인원 도구 | VM 또는 컨테이너 | 로컬 학습/개발, PoC |
| **k3d** | Docker 위에 k3s 실행 도구 | Docker 컨테이너 | k3s 기반 로컬/테스트 |
| **k3s** | 경량 Kubernetes 배포판 | 베어메탈, VM, 엣지 디바이스 | 엣지/온프레미스/소형 클러스터 |

### 어떤 상황에서 무엇을 쓸까?

**1) 빠른 테스트/CI용 클러스터가 필요할 때**
- **추천:** `kind`
- 이유: 설치/삭제가 빠르고, CI에서 스크립트로 사용하기 좋음

**2) 로컬에서 쿠버네티스를 처음 공부할 때**
- **추천:** `minikube`
- 이유: Dashboard, Ingress, LoadBalancer 등 **"쿠버네티스에 있는 거의 모든 것"**을 로컬에서 체험 가능

**3) k3s 기반 환경을 로컬에서 실험하고 싶을 때**
- **추천:** `k3d`
- 이유: k3s를 Docker로 쉽게 돌려볼 수 있음 (특히 k3s 기반 운영 환경을 미리 연습할 때)

**4) 실제 엣지/온프레미스 소형 클러스터를 만들 때**
- **추천:** `k3s`
- 이유: 리소스가 적은 환경(라즈베리파이, 소형 서버 등)에 최적화된 경량 배포판

---

## 7. 실전 선택 가이드 (개발자 관점)

### 백엔드 개발자가 로컬에서 API를 테스트하고 싶을 때

- 쿠버네티스에 **처음 입문**하는 단계라면: `minikube`
  - Ingress, Service, Deployment, ConfigMap 등을 한 번씩 써보기에 좋음
- 이미 EKS/GKE 등 실제 클러스터를 쓰고 있고, **로컬 재현용**이 필요하다면: `kind` 또는 `k3d`
  - CI 파이프라인에 넣기에는 `kind`가 조금 더 익숙한 선택지

### 인프라/플랫폼 엔지니어 관점

- **테스트/CI:** `kind`
- **k3s 기반 엣지/온프레미스 설계:** 실제 운영은 `k3s`, 로컬 실험은 `k3d`
- **교육/워크샵:** `minikube` (대부분의 기능을 한 번에 보여주기 좋음)

---

## 마무리

**핵심 포인트:**

- **kind / minikube / k3d**는 주로 "어디에 Kubernetes를 띄울 것인가"에 초점이 맞춰진 **도구**입니다.
- **k3s**는 "어떤 Kubernetes를 띄울 것인가"에 초점이 맞춰진 **경량 배포판**입니다.
- 로컬 개발과 CI에는 `kind`/`minikube`/`k3d`를, 실제 엣지/온프레미스 환경에는 `k3s`를 고려하면 됩니다.

쿠버네티스를 공부할 때 처음부터 모든 도구를 다 써볼 필요는 없습니다.  
현재 목표(학습, 로컬 개발, CI, 엣지 운영 등)에 맞춰 **하나를 먼저 깊게 써본 뒤**, 필요에 따라 다른 도구를 비교해보는 것이 좋습니다. 🚀

다음 글에서는 마이크로서비스 아키텍처에서 **설정 관리**를 어떻게 할지, 특히 **Spring Cloud Config Server와 Git 브랜치 전략**을 멀티 모듈 MSA 환경에서 어떻게 구성하는지 정리해보겠습니다.

