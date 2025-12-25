---
layout: post
title: "Kubernetes 환경에서 Redis 운영 전략: Pod vs Docker Compose"
date: 2025-12-25
categories: [kubernetes, redis]
tags: [Kubernetes, Redis, Pod, Docker Compose, 인프라전략, 운영전략]
---

# Kubernetes 환경에서 Redis 운영 전략: Pod vs Docker Compose

이전 글에서 Spring Cloud Bus를 다뤘는데, 이번에는 Kubernetes 환경에서 **Redis를 어떻게 운영할지**에 대한 고민을 정리해보겠습니다.

Kubernetes 클러스터를 사용하면서 애플리케이션은 Pod로 배포하는데, **Redis 같은 인프라 컴포넌트는 Pod로 돌려야 할까요, 아니면 Docker Compose로 별도로 운영해야 할까요?**  
이 질문은 특히 **개발 환경**이나 **소규모 프로덕션 환경**에서 자주 마주치는 딜레마입니다.

이 글에서는 Kubernetes 환경에서 Redis를 **Pod로 운영하는 방법**과 **Docker Compose로 운영하는 방법**을 비교하고, 각각의 장단점과 사용 시나리오를 정리해보겠습니다.

---

## 1. 문제 상황: Kubernetes 환경에서의 Redis 운영

### 일반적인 시나리오

**상황:**
- Kubernetes 클러스터에 애플리케이션 Pod들이 배포되어 있음
- Redis를 캐시나 세션 저장소로 사용해야 함
- 다른 DB들(PostgreSQL, MySQL 등)은 Docker Compose로 운영 중

**고민:**
- Redis도 **Kubernetes Pod로 배포**해야 할까?
- 아니면 **Docker Compose로 별도 운영**해야 할까?
- 두 방식을 **혼용**해도 될까?

---

## 2. 방법 1: Redis를 Kubernetes Pod로 운영

### 개념

**Redis를 Kubernetes의 StatefulSet 또는 Deployment로 배포**하는 방식입니다.

```yaml
# redis-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
spec:
  serviceName: redis
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        ports:
        - containerPort: 6379
        volumeMounts:
        - name: redis-data
          mountPath: /data
  volumeClaimTemplates:
  - metadata:
      name: redis-data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 10Gi
---
apiVersion: v1
kind: Service
metadata:
  name: redis
spec:
  selector:
    app: redis
  ports:
  - port: 6379
    targetPort: 6379
```

### 장점

- ✅ **통합 관리**: 애플리케이션과 Redis가 같은 Kubernetes 클러스터에서 관리됨
- ✅ **네트워크 단순화**: Service를 통해 `redis:6379`로 접근 가능 (DNS 기반)
- ✅ **리소스 관리**: Kubernetes의 Resource Quota, LimitRange로 리소스 제어 가능
- ✅ **모니터링 통합**: Prometheus, Grafana 등으로 애플리케이션과 Redis를 함께 모니터링
- ✅ **자동 복구**: Pod가 죽으면 Kubernetes가 자동으로 재시작
- ✅ **스케일링**: Horizontal Pod Autoscaler(HPA)로 자동 스케일링 가능 (단일 인스턴스는 제한적)

### 단점

- ❌ **영속성 관리 복잡도**: PVC(PersistentVolumeClaim) 설정 필요, 백업/복구 전략 수립 필요
- ❌ **클러스터 리소스 소비**: Kubernetes 노드의 CPU/메모리를 사용
- ❌ **운영 복잡도**: Redis 클러스터 구성 시 복잡도 증가 (Sentinel, Cluster 모드)
- ❌ **노드 장애 영향**: 노드가 다운되면 Redis Pod도 영향받을 수 있음 (다중 노드 + 스토리지 고가용성 필요)

---

## 3. 방법 2: Redis를 Docker Compose로 운영

### 개념

**Redis를 Kubernetes 클러스터 외부에서 Docker Compose로 별도 운영**하는 방식입니다.

```yaml
# docker-compose.yml
version: '3.8'

services:
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    command: redis-server --appendonly yes
    restart: unless-stopped

  postgres:
    image: postgres:15
    ports:
      - "5432:5432"
    environment:
      POSTGRES_DB: mydb
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
    volumes:
      - postgres-data:/var/lib/postgresql/data
    restart: unless-stopped

volumes:
  redis-data:
  postgres-data:
```

**Kubernetes에서 접근:**
```yaml
# 애플리케이션 Pod의 환경 변수
env:
- name: REDIS_HOST
  value: "host.docker.internal"  # 또는 실제 호스트 IP
- name: REDIS_PORT
  value: "6379"
```

### 장점

- ✅ **운영 단순성**: Docker Compose로 간단하게 시작/중지 가능
- ✅ **개발 환경 친화적**: 로컬 개발 환경과 동일한 방식으로 운영
- ✅ **리소스 분리**: Kubernetes 클러스터 리소스를 사용하지 않음
- ✅ **기존 인프라 활용**: 다른 DB들과 함께 Docker Compose로 통합 관리 가능
- ✅ **백업/복구 용이**: 호스트 파일 시스템에 직접 접근하여 백업 가능

### 단점

- ❌ **네트워크 복잡도**: Kubernetes 클러스터 외부에서 접근해야 하므로 네트워크 설정 필요
- ❌ **고가용성 부족**: 단일 인스턴스로 운영 시 장애 대응 어려움
- ❌ **통합 관리 어려움**: Kubernetes의 모니터링, 로깅 시스템과 분리됨
- ❌ **스케일링 제한**: 수동으로 스케일링해야 함
- ❌ **환경 불일치**: 개발/프로덕션 환경이 다를 수 있음

---

## 4. 방법 3: Managed Redis 서비스 활용

### 개념

**클라우드 제공 업체의 Managed Redis 서비스**(예: AWS ElastiCache, Azure Cache for Redis, GCP Memorystore)를 사용하는 방식입니다.

### 장점

- ✅ **운영 부담 최소화**: 백업, 패치, 모니터링을 클라우드 제공 업체가 관리
- ✅ **고가용성**: 자동 failover, Multi-AZ 지원
- ✅ **성능 최적화**: 클라우드 제공 업체가 최적화된 인프라 제공
- ✅ **보안**: VPC 내부 네트워크, 암호화 지원

### 단점

- ❌ **비용**: 사용량에 따라 비용 발생
- ❌ **벤더 종속성**: 특정 클라우드 제공 업체에 종속
- ❌ **커스터마이징 제한**: 설정 옵션이 제한적일 수 있음

---

## 5. 상황별 추천 전략

### 개발 환경 (Development)

**추천: Docker Compose**

**이유:**
- 개발자가 로컬에서 쉽게 실행/중지 가능
- Kubernetes 클러스터 리소스를 절약
- 설정 변경이 간단함

```yaml
# docker-compose.dev.yml
services:
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - ./redis-data:/data
```

---

### 스테이징 환경 (Staging)

**추천: Kubernetes Pod (단일 인스턴스)**

**이유:**
- 프로덕션 환경과 유사한 구성으로 테스트 가능
- Kubernetes의 Service Discovery 활용
- 모니터링 통합

```yaml
# redis-staging.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
spec:
  replicas: 1
  template:
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
```

---

### 프로덕션 환경 (Production)

**추천: Managed Redis 서비스 또는 Kubernetes Pod (고가용성 구성)**

**상황별 선택:**

**1) 소규모 프로덕션 (트래픽이 적고, 비용이 중요할 때)**
- **Kubernetes Pod + Redis Sentinel** 또는 **Redis Cluster**
- PVC로 영속성 보장
- 다중 노드에 배포하여 고가용성 확보

**2) 중대규모 프로덕션 (운영 부담을 줄이고 싶을 때)**
- **Managed Redis 서비스** (AWS ElastiCache, Azure Cache 등)
- 자동 백업, 모니터링, 고가용성 제공

---

## 6. 실전 비교: Pod vs Docker Compose

### 네트워크 접근 방식

**Kubernetes Pod:**
```yaml
# 애플리케이션에서 Redis 접근
spring:
  redis:
    host: redis  # Service 이름으로 DNS 자동 해석
    port: 6379
```

**Docker Compose:**
```yaml
# 애플리케이션에서 Redis 접근
spring:
  redis:
    host: ${REDIS_HOST:host.docker.internal}  # 환경 변수 필요
    port: 6379
```

---

### 영속성 관리

**Kubernetes Pod:**
```yaml
# PVC를 통한 영속성
volumeClaimTemplates:
- metadata:
    name: redis-data
  spec:
    accessModes: [ "ReadWriteOnce" ]
    resources:
      requests:
        storage: 10Gi
```

**Docker Compose:**
```yaml
# 호스트 볼륨을 통한 영속성
volumes:
  redis-data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /var/lib/redis-data
```

---

### 모니터링 통합

**Kubernetes Pod:**
- Prometheus가 Pod 메트릭을 자동 수집
- Grafana 대시보드에서 애플리케이션과 Redis를 함께 모니터링
- Kubernetes Events로 상태 추적

**Docker Compose:**
- 별도로 Prometheus Exporter 설정 필요
- 호스트 메트릭과 컨테이너 메트릭을 분리해서 수집

---

## 7. 하이브리드 접근: 상황별 선택

### 권장 구성

**개발 환경:**
- Redis: Docker Compose
- PostgreSQL: Docker Compose
- 애플리케이션: Kubernetes Pod (또는 로컬 실행)

**스테이징 환경:**
- Redis: Kubernetes Pod (단일 인스턴스)
- PostgreSQL: Kubernetes Pod 또는 Managed DB
- 애플리케이션: Kubernetes Pod

**프로덕션 환경:**
- Redis: Managed Redis 서비스 (또는 Kubernetes Pod + 고가용성)
- PostgreSQL: Managed DB 서비스
- 애플리케이션: Kubernetes Pod

---

## 8. Redis 고가용성 구성 (Kubernetes Pod)

### Redis Sentinel 구성

```yaml
# redis-master.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis-master
spec:
  serviceName: redis-master
  replicas: 1
  template:
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        command:
          - redis-server
          - --appendonly
          - "yes"
---
# redis-sentinel.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis-sentinel
spec:
  serviceName: redis-sentinel
  replicas: 3  # Sentinel은 홀수 개 권장
  template:
    spec:
      containers:
      - name: sentinel
        image: redis:7-alpine
        command:
          - redis-sentinel
          - /etc/redis/sentinel.conf
```

### Redis Cluster 구성

```yaml
# redis-cluster.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis-cluster
spec:
  serviceName: redis-cluster
  replicas: 6  # 최소 3 master + 3 replica
  template:
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        command:
          - redis-server
          - --cluster-enabled
          - "yes"
          - --cluster-config-file
          - /data/nodes.conf
```

**주의사항:**
- Redis Cluster는 복잡도가 높으므로, **Managed 서비스**를 고려하는 것이 좋습니다.

---

## 9. 마이그레이션 전략

### Docker Compose → Kubernetes Pod로 마이그레이션

**단계:**
1. **데이터 백업**: Docker Compose의 Redis 데이터를 덤프
2. **Kubernetes Pod 배포**: StatefulSet으로 Redis 배포
3. **데이터 복원**: 백업한 데이터를 PVC에 복원
4. **애플리케이션 설정 변경**: Redis 호스트를 Service 이름으로 변경
5. **검증**: 애플리케이션이 정상 동작하는지 확인
6. **Docker Compose 중지**: 기존 Redis 컨테이너 중지

---

## 10. 실전 고려사항

### 비용 관점

- **Kubernetes Pod**: 클러스터 노드 비용 (이미 지불 중)
- **Docker Compose**: 별도 서버 비용 (또는 기존 서버 활용)
- **Managed 서비스**: 사용량 기반 비용 (예: AWS ElastiCache)

### 운영 복잡도

- **Kubernetes Pod**: 초기 설정 복잡, 이후 자동화 가능
- **Docker Compose**: 초기 설정 간단, 스케일링/고가용성은 수동
- **Managed 서비스**: 설정 간단, 운영 부담 최소

### 성능

- **Kubernetes Pod**: 네트워크 오버헤드 (Pod 간 통신)
- **Docker Compose**: 호스트 네트워크 사용 시 오버헤드 적음
- **Managed 서비스**: 최적화된 네트워크 및 하드웨어

---

## 마무리

**핵심 포인트:**

- **개발 환경**에서는 **Docker Compose**가 단순하고 효율적입니다.
- **스테이징 환경**에서는 **Kubernetes Pod**로 프로덕션과 유사한 환경을 구성하는 것이 좋습니다.
- **프로덕션 환경**에서는 **규모와 운영 역량**에 따라 선택:
  - 소규모: Kubernetes Pod + 고가용성 구성
  - 중대규모: Managed Redis 서비스
- **하이브리드 접근**도 가능합니다: 환경별로 다른 방식을 사용

Redis 운영 전략은 **비용, 운영 복잡도, 고가용성 요구사항**을 종합적으로 고려하여 선택해야 합니다.  
특히 프로덕션 환경에서는 **데이터 손실 방지**와 **고가용성**이 가장 중요하므로, 충분한 테스트와 모니터링을 구축한 후 결정하는 것이 좋습니다. 🚀

다음 글에서는 **Redis의 영속성 전략**(RDB, AOF)과 **백업/복구 방법**을 정리해보겠습니다.

