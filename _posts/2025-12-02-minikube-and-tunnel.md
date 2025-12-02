---
layout: post
title: "Minikube 개념과 터널 기능 정리"
date: 2025-12-02
categories: [kubernetes]
tags: [Minikube, Kubernetes, 터널, 로컬개발, K8S, 컨테이너]
---

# Minikube 개념과 터널 기능 정리

Kubernetes는 컨테이너 오케스트레이션의 표준으로 자리 잡았지만, 학습이나 개발을 위해 전체 클러스터를 구축하는 것은 부담스러울 수 있습니다. **Minikube**는 로컬 환경에서 단일 노드 Kubernetes 클러스터를 실행할 수 있게 해주는 경량화된 도구입니다. 이번 포스트에서는 Minikube의 개념, 설치 방법, 그리고 터널 기능을 포함한 주요 기능들을 정리해보겠습니다.

## Minikube란?

**Minikube**는 로컬 머신에서 단일 노드 Kubernetes 클러스터를 실행할 수 있게 해주는 오픈소스 도구입니다. 개발자들이 Kubernetes를 학습하고, 애플리케이션을 테스트하며, 로컬에서 개발 환경을 구축하는 데 유용합니다.

### Minikube의 특징

- **경량화**: 단일 노드 클러스터로 리소스 사용량이 적음
- **빠른 시작**: 간단한 명령어로 클러스터 시작/중지
- **다양한 드라이버 지원**: VirtualBox, Docker, Hyper-V 등
- **Kubernetes 기능 지원**: 대부분의 Kubernetes 기능을 로컬에서 테스트 가능
- **애드온 지원**: Dashboard, Ingress 등 다양한 애드온 제공

## Minikube 설치

### macOS

```bash
# Homebrew를 사용한 설치
brew install minikube

# 또는 직접 다운로드
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-darwin-amd64
sudo install minikube-darwin-amd64 /usr/local/bin/minikube
```

### Linux

```bash
# 직접 다운로드
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

### Windows

```powershell
# Chocolatey를 사용한 설치
choco install minikube

# 또는 직접 다운로드
# https://storage.googleapis.com/minikube/releases/latest/minikube-installer.exe
```

## Minikube 시작하기

### 기본 시작

```bash
# Minikube 클러스터 시작
minikube start

# 특정 드라이버 지정
minikube start --driver=virtualbox
minikube start --driver=docker
minikube start --driver=hyperkit
```

### 리소스 설정

```bash
# CPU와 메모리 할당량 지정
minikube start --cpus=4 --memory=8192

# 디스크 크기 지정
minikube start --disk-size=20g
```

### Kubernetes 버전 지정

```bash
# 특정 Kubernetes 버전 사용
minikube start --kubernetes-version=v1.28.0
```

## Minikube 주요 명령어

### 클러스터 관리

```bash
# 클러스터 상태 확인
minikube status

# 클러스터 중지
minikube stop

# 클러스터 삭제
minikube delete

# 클러스터 일시중지
minikube pause

# 클러스터 재개
minikube unpause
```

### 클러스터 정보

```bash
# 클러스터 IP 확인
minikube ip

# Kubernetes 대시보드 열기
minikube dashboard

# 서비스 URL 확인
minikube service <service-name> --url
```

### 설정 관리

```bash
# 설정 확인
minikube config view

# 설정 변경
minikube config set cpus 4
minikube config set memory 8192

# 설정 삭제
minikube config unset <key>
```

## Minikube 터널 기능

### 터널이 필요한 이유

Kubernetes에서 `LoadBalancer` 타입의 Service를 사용하면, 클라우드 환경에서는 자동으로 외부 IP가 할당됩니다. 하지만 로컬 환경(Minikube)에서는 실제 로드 밸런서가 없기 때문에 `LoadBalancer` 타입의 Service가 `pending` 상태로 남아있게 됩니다.

**문제 상황:**
```bash
$ kubectl get svc
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
my-service   LoadBalancer   10.96.0.1       <pending>     80:30000/TCP   5m
```

### 터널 기능 사용법

**Minikube 터널**은 로컬 환경에서 `LoadBalancer` 타입의 Service를 외부에서 접근할 수 있게 해주는 기능입니다.

```bash
# 터널 시작 (별도 터미널에서 실행)
minikube tunnel

# 터널은 백그라운드에서 실행되며, Service의 EXTERNAL-IP를 할당합니다
```

**터널 실행 후:**
```bash
$ kubectl get svc
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
my-service   LoadBalancer   10.96.0.1       127.0.0.1     80:30000/TCP   5m
```

이제 `http://127.0.0.1` 또는 `http://localhost`로 서비스에 접근할 수 있습니다.

### 터널 동작 원리

1. **Minikube 터널 실행**
   - 호스트 머신의 네트워크 인터페이스에 라우팅 규칙 추가
   - LoadBalancer Service의 IP를 호스트의 로컬 IP로 매핑

2. **트래픽 흐름**
   ```
   외부 요청 (localhost:80)
       ↓
   호스트 머신의 라우팅 규칙
       ↓
   Minikube VM으로 트래픽 전달
       ↓
   Kubernetes Service
       ↓
   Pod
   ```

3. **터널 종료**
   - `Ctrl+C`로 터널 종료
   - 라우팅 규칙이 자동으로 제거됨

### 터널 사용 시 주의사항

1. **권한 요구**
   - 터널은 네트워크 라우팅 규칙을 변경하므로 관리자 권한이 필요할 수 있습니다
   - macOS/Linux: `sudo minikube tunnel`

2. **터널 유지**
   - 터널은 별도 터미널에서 계속 실행되어야 합니다
   - 터널을 종료하면 EXTERNAL-IP가 다시 `<pending>` 상태가 됩니다

3. **포트 충돌**
   - 호스트 머신에서 이미 사용 중인 포트와 충돌할 수 있습니다
   - Service의 포트를 변경하거나 호스트의 다른 서비스를 중지해야 합니다

## Minikube 애드온

Minikube는 다양한 애드온을 제공하여 클러스터 기능을 확장할 수 있습니다.

### 애드온 목록 확인

```bash
minikube addons list
```

### 주요 애드온

#### 1. Dashboard

```bash
# Kubernetes 대시보드 활성화
minikube addons enable dashboard

# 대시보드 열기
minikube dashboard
```

#### 2. Ingress

```bash
# NGINX Ingress Controller 활성화
minikube addons enable ingress

# Ingress Controller 확인
kubectl get pods -n ingress-nginx
```

#### 3. Metrics Server

```bash
# Metrics Server 활성화
minikube addons enable metrics-server

# 리소스 사용량 확인
kubectl top nodes
kubectl top pods
```

#### 4. Storage Provisioner

```bash
# 기본 스토리지 프로비저너 활성화
minikube addons enable storage-provisioner
```

### 애드온 관리

```bash
# 애드온 활성화
minikube addons enable <addon-name>

# 애드온 비활성화
minikube addons disable <addon-name>

# 애드온 상태 확인
minikube addons list
```

## 실전 예제: Minikube에서 애플리케이션 배포

### 1. Minikube 시작

```bash
minikube start --driver=docker --cpus=4 --memory=8192
```

### 2. 애플리케이션 배포

```bash
# Deployment 생성
kubectl create deployment nginx --image=nginx

# Service 생성 (LoadBalancer 타입)
kubectl expose deployment nginx --type=LoadBalancer --port=80
```

### 3. 터널 시작

```bash
# 별도 터미널에서 실행
minikube tunnel
```

### 4. 서비스 접근

```bash
# Service 상태 확인
kubectl get svc nginx

# 브라우저에서 http://127.0.0.1 접근
# 또는 curl 사용
curl http://127.0.0.1
```

## Minikube vs 다른 로컬 Kubernetes 도구

### Minikube

**장점:**
- 안정적이고 널리 사용됨
- 다양한 드라이버 지원
- 풍부한 문서와 커뮤니티

**단점:**
- VM 기반이라 리소스 사용량이 상대적으로 큼
- 시작 시간이 다소 느림

### kind (Kubernetes in Docker)

**장점:**
- Docker 컨테이너 기반으로 가벼움
- 빠른 시작 시간

**단점:**
- Docker 환경에서만 동작
- 일부 기능 제한

### k3d (k3s in Docker)

**장점:**
- 매우 가벼움
- 빠른 시작 시간

**단점:**
- k3s 기반으로 일부 Kubernetes 기능 제한

## 모범 사례

### 개발 워크플로우

1. **로컬 개발**
   - Minikube에서 애플리케이션 개발 및 테스트
   - 터널을 사용하여 외부 접근 테스트

2. **CI/CD 통합**
   - CI 파이프라인에서 Minikube를 사용한 통합 테스트
   - 배포 전 검증

3. **학습 환경**
   - Kubernetes 기능 학습
   - 다양한 리소스 실습

### 리소스 관리

```bash
# 개발 중에는 최소 리소스로 시작
minikube start --cpus=2 --memory=4096

# 테스트가 필요할 때 리소스 증가
minikube stop
minikube start --cpus=4 --memory=8192
```

### 정리

```bash
# 사용하지 않을 때는 클러스터 중지
minikube stop

# 완전히 삭제하려면
minikube delete
```

**로컬 개발 환경에서 Minikube를 사용하여 Kubernetes 기반 마이크로서비스 아키텍처를 테스트하는 방법을 공부하고 있으며, 적용할 예정입니다.** Minikube의 터널 기능을 활용하여 LoadBalancer 타입의 Service를 로컬에서 테스트할 수 있어, 실제 클라우드 환경 배포 전에 충분한 검증이 가능합니다.

---

다음 포스트에서는 **Kafka와 Elasticsearch 연동 가이드**에 대해 다루겠습니다. 실시간 데이터 파이프라인을 구축하여 이벤트 스트리밍과 검색 기능을 효율적으로 구현하는 방법을 정리해보겠습니다. 실전 예제와 함께 알아봐요! 📊

