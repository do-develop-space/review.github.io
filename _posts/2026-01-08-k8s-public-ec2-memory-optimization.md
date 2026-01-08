---
layout: post
title: "Kubernetes Public EC2 메모리 최적화: t3.medium 4GB 환경에서 여러 Pod 운영하기"
date: 2026-01-08
categories: [kubernetes, devops, optimization]
tags: [Kubernetes, 메모리최적화, 리소스관리, EC2, t3.medium, JVM튜닝, Pod최적화]
---

# Kubernetes Public EC2 메모리 최적화: t3.medium 4GB 환경에서 여러 Pod 운영하기

이전 글에서 Master Node Pod의 Toleration 설정을 통해 Worker Node 메모리를 보호하는 방법을 다뤘습니다. 이번 글에서는 **Public EC2 인스턴스(t3.medium, 2코어 4GB)**에서 여러 Pod를 운영할 때 발생하는 메모리 부족 문제를 해결하는 방법을 정리해보겠습니다.

Public으로 배포되는 Pod들(control-plane, cloud 서버, frontend, mysql, redis, elasticsearch, kafka)이 한 노드에 모두 배포되어 메모리가 부족한 상황에서, requests/limits를 제거하고 JAVA_OPTS만 설정한 현재 상태를 개선하는 최적화 전략을 살펴보겠습니다.

---

## 1. 문제 상황

### 1.1 현재 환경

**EC2 인스턴스 사양:**
- 인스턴스 타입: `t3.medium`
- CPU: 2 vCPU
- 메모리: 4GB
- 용도: Public 서비스 배포

**배포된 Pod 목록:**
- control-plane 컴포넌트
- cloud 서버 (Spring Boot 애플리케이션)
- frontend (웹 프론트엔드)
- mysql (데이터베이스)
- redis (캐시)
- elasticsearch (검색 엔진)
- kafka (메시지 브로커)

**현재 설정:**
- requests와 limits를 전부 제거
- JAVA_OPTS만 설정한 상태
- 메모리 부족으로 인한 Pod 재시작 빈발

### 1.2 메모리 부족 문제

**발생하는 문제:**
- Pod들이 OOMKilled (Out Of Memory)로 종료
- 메모리 부족으로 인한 성능 저하
- Pod 재시작으로 인한 서비스 중단
- 시스템 메모리 부족으로 인한 노드 불안정

**메모리 사용 현황 (예상):**
```
EC2 인스턴스: 4GB
├── 시스템 (OS, kubelet 등): ~500MB
├── control-plane: ~200MB
├── cloud 서버: ~1GB (JAVA_OPTS 설정)
├── frontend: ~100MB
├── mysql: ~500MB
├── redis: ~200MB
├── elasticsearch: ~1GB
└── kafka: ~500MB

총 사용량: ~4GB (한계치 초과)
```

---

## 2. 메모리 최적화 전략

### 2.1 전략 개요

**최적화 접근 방법:**
1. **JVM 힙 메모리 최적화**: Java 애플리케이션의 힙 크기 조정
2. **리소스 requests/limits 설정**: 각 Pod의 메모리 제한 설정
3. **컨테이너별 메모리 최적화**: 서비스별 특성에 맞는 메모리 설정
4. **불필요한 서비스 제거 또는 외부화**: 일부 서비스를 별도 인스턴스로 분리

### 2.2 최적화 우선순위

**1순위: JVM 힙 메모리 최적화**
- Java 애플리케이션(cloud 서버)의 힙 크기 조정
- JAVA_OPTS 최적화

**2순위: 리소스 requests/limits 설정**
- 각 Pod의 메모리 제한 설정
- OOMKilled 방지

**3순위: 서비스별 메모리 최적화**
- 각 서비스의 특성에 맞는 메모리 설정
- 불필요한 메모리 사용 최소화

---

## 3. JVM 힙 메모리 최적화

### 3.1 현재 JAVA_OPTS 분석

**일반적인 JAVA_OPTS 설정:**

```yaml
env:
- name: JAVA_OPTS
  value: "-Xms512m -Xmx1024m -XX:+UseG1GC"
```

**문제점:**
- 힙 메모리가 너무 큼 (1GB)
- 4GB 환경에서 다른 서비스와 메모리 경쟁
- 메타스페이스, 스레드 스택 등 추가 메모리 사용

### 3.2 최적화된 JAVA_OPTS 설정

**최적화 전략:**

```yaml
env:
- name: JAVA_OPTS
  value: >
    -Xms256m
    -Xmx512m
    -XX:MetaspaceSize=128m
    -XX:MaxMetaspaceSize=256m
    -XX:+UseG1GC
    -XX:MaxGCPauseMillis=200
    -XX:+UseStringDeduplication
    -XX:+OptimizeStringConcat
    -Djava.awt.headless=true
    -Dfile.encoding=UTF-8
```

**설정 설명:**
- `-Xms256m -Xmx512m`: 힙 메모리 256MB~512MB (기존 1GB에서 절반으로 감소)
- `-XX:MetaspaceSize=128m`: 메타스페이스 초기 크기
- `-XX:MaxMetaspaceSize=256m`: 메타스페이스 최대 크기
- `-XX:+UseG1GC`: G1 가비지 컬렉터 사용 (낮은 메모리 환경에 적합)
- `-XX:MaxGCPauseMillis=200`: GC 일시정지 시간 목표 (200ms)
- `-XX:+UseStringDeduplication`: 문자열 중복 제거로 메모리 절약
- `-XX:+OptimizeStringConcat`: 문자열 연결 최적화

### 3.3 Spring Boot 애플리케이션 최적화

**application.yaml 설정:**

```yaml
spring:
  jpa:
    properties:
      hibernate:
        jdbc:
          batch_size: 20
        order_inserts: true
        order_updates: true
  datasource:
    hikari:
      maximum-pool-size: 5  # 커넥션 풀 크기 감소
      minimum-idle: 2
```

**메모리 절약 효과:**
- 커넥션 풀 크기 감소로 메모리 사용량 감소
- 배치 처리로 메모리 효율 향상

---

## 4. 리소스 requests/limits 설정

### 4.1 메모리 제한 설정 전략

**전체 메모리 배분 (4GB 기준):**

```
시스템 (OS, kubelet 등): 500MB
여유 공간 (버퍼): 300MB
사용 가능 메모리: 3.2GB

Pod별 메모리 배분:
├── control-plane: 200MB
├── cloud 서버: 600MB (JVM 힙 512MB + 오버헤드)
├── frontend: 100MB
├── mysql: 400MB
├── redis: 200MB
├── elasticsearch: 800MB
└── kafka: 500MB

총: 2.8GB (여유 공간 400MB)
```

### 4.2 Cloud 서버 Pod 설정

**Deployment 예시:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cloud-server
spec:
  replicas: 1
  template:
    spec:
      containers:
      - name: cloud-server
        image: cloud-server:latest
        env:
        - name: JAVA_OPTS
          value: >
            -Xms256m
            -Xmx512m
            -XX:MetaspaceSize=128m
            -XX:MaxMetaspaceSize=256m
            -XX:+UseG1GC
            -XX:MaxGCPauseMillis=200
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "600Mi"  # 힙 512MB + 오버헤드 88MB
            cpu: "1000m"
```

**설정 설명:**
- `requests.memory: 512Mi`: 스케줄러가 보장하는 최소 메모리
- `limits.memory: 600Mi`: 최대 메모리 (힙 512MB + 메타스페이스 256MB + 오버헤드)
- JVM 오버헤드: 힙 메모리의 약 10-20% 추가 필요

### 4.3 MySQL Pod 설정

**MySQL 최적화:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  template:
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
        - name: MYSQL_INNODB_BUFFER_POOL_SIZE
          value: "256M"  # InnoDB 버퍼 풀 크기 감소
        resources:
          requests:
            memory: "300Mi"
            cpu: "200m"
          limits:
            memory: "400Mi"
            cpu: "500m"
```

**MySQL 설정 최적화:**
- `innodb_buffer_pool_size`: 256MB (기본값보다 감소)
- `max_connections`: 50 (기본값보다 감소)
- `query_cache_size`: 비활성화 (MySQL 8.0에서는 제거됨)

### 4.4 Redis Pod 설정

**Redis 최적화:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
spec:
  template:
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        command:
        - redis-server
        - --maxmemory
        - "150mb"  # 최대 메모리 150MB
        - --maxmemory-policy
        - "allkeys-lru"  # LRU 정책으로 메모리 관리
        resources:
          requests:
            memory: "150Mi"
            cpu: "100m"
          limits:
            memory: "200Mi"
            cpu: "200m"
```

**Redis 설정 최적화:**
- `maxmemory`: 150MB로 제한
- `maxmemory-policy`: `allkeys-lru` (LRU 정책으로 오래된 키 제거)
- Alpine 이미지 사용으로 메모리 사용량 감소

### 4.5 Elasticsearch Pod 설정

**Elasticsearch 최적화:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: elasticsearch
spec:
  template:
    spec:
      containers:
      - name: elasticsearch
        image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
        env:
        - name: ES_JAVA_OPTS
          value: "-Xms256m -Xmx512m"  # 힙 메모리 512MB
        - name: discovery.type
          value: "single-node"
        resources:
          requests:
            memory: "600Mi"
            cpu: "500m"
          limits:
            memory: "800Mi"
            cpu: "1000m"
```

**Elasticsearch 설정 최적화:**
- 힙 메모리: 512MB (기본값보다 감소)
- 단일 노드 모드로 운영
- 인덱스 샤드 수 최소화

### 4.6 Kafka Pod 설정

**Kafka 최적화:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kafka
spec:
  template:
    spec:
      containers:
      - name: kafka
        image: confluentinc/cp-kafka:latest
        env:
        - name: KAFKA_HEAP_OPTS
          value: "-Xmx256m -Xms256m"  # 힙 메모리 256MB
        - name: KAFKA_JVM_PERFORMANCE_OPTS
          value: >
            -XX:+UseG1GC
            -XX:MaxGCPauseMillis=200
        resources:
          requests:
            memory: "300Mi"
            cpu: "300m"
          limits:
            memory: "500Mi"
            cpu: "500m"
```

**Kafka 설정 최적화:**
- 힙 메모리: 256MB
- 로그 세그먼트 크기 감소
- 브로커 수 최소화

---

## 5. 컨테이너별 추가 최적화

### 5.1 Frontend Pod 최적화

**Frontend (Nginx 등) 최적화:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  template:
    spec:
      containers:
      - name: frontend
        image: nginx:alpine
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "100Mi"
            cpu: "100m"
```

**최적화 포인트:**
- Alpine 이미지 사용으로 메모리 사용량 감소
- 정적 파일은 CDN으로 오프로드 고려

### 5.2 Control Plane 컴포넌트 최적화

**Control Plane 리소스 제한:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kube-controller-manager
  namespace: kube-system
spec:
  template:
    spec:
      containers:
      - name: kube-controller-manager
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "200Mi"
            cpu: "200m"
```

---

## 6. 전체 리소스 요약

### 6.1 최적화된 메모리 배분

**최종 메모리 배분:**

| Pod | Requests | Limits | 설명 |
|-----|----------|--------|------|
| **시스템** | - | 500MB | OS, kubelet 등 |
| **control-plane** | 128MB | 200MB | Control Plane 컴포넌트 |
| **cloud 서버** | 512MB | 600MB | JVM 힙 512MB |
| **frontend** | 64MB | 100MB | Nginx |
| **mysql** | 300MB | 400MB | InnoDB 버퍼 256MB |
| **redis** | 150MB | 200MB | 최대 메모리 150MB |
| **elasticsearch** | 600MB | 800MB | 힙 512MB |
| **kafka** | 300MB | 500MB | 힙 256MB |
| **여유 공간** | - | 400MB | 버퍼 |
| **총합** | ~1.9GB | ~3.7GB | 4GB 환경 내 |

### 6.2 CPU 배분

**CPU 배분 (2 vCPU 기준):**

| Pod | Requests | Limits |
|-----|----------|--------|
| **control-plane** | 100m | 200m |
| **cloud 서버** | 500m | 1000m |
| **frontend** | 50m | 100m |
| **mysql** | 200m | 500m |
| **redis** | 100m | 200m |
| **elasticsearch** | 500m | 1000m |
| **kafka** | 300m | 500m |
| **총합** | ~1.75 CPU | ~3.5 CPU |

---

## 7. 모니터링 및 검증

### 7.1 메모리 사용량 모니터링

**Pod별 메모리 사용량 확인:**

```bash
# Pod별 메모리 사용량 확인
kubectl top pods

# 특정 Pod의 상세 정보 확인
kubectl describe pod <pod-name>

# 메모리 사용량 추이 확인
kubectl top pods --containers
```

### 7.2 노드 메모리 확인

**노드 리소스 확인:**

```bash
# 노드 리소스 사용량 확인
kubectl top nodes

# 노드 상세 정보 확인
kubectl describe node <node-name>

# 메모리 압력 확인
kubectl get events --field-selector reason=OOMKilling
```

### 7.3 OOMKilled 방지 확인

**OOMKilled 이벤트 모니터링:**

```bash
# OOMKilled 이벤트 확인
kubectl get events --sort-by='.lastTimestamp' | grep OOMKilled

# Pod 재시작 횟수 확인
kubectl get pods | grep -E "Restarts|OOMKilled"
```

---

## 8. 추가 최적화 전략

### 8.1 서비스 외부화 고려

**외부화 가능한 서비스:**
- **MySQL**: RDS 또는 별도 인스턴스로 이전
- **Redis**: ElastiCache 또는 별도 인스턴스로 이전
- **Elasticsearch**: OpenSearch Service 또는 별도 인스턴스로 이전
- **Kafka**: MSK(Managed Streaming for Kafka) 또는 별도 인스턴스로 이전

**장점:**
- Public EC2의 메모리 부담 감소
- 관리형 서비스의 안정성 향상
- 확장성 향상

### 8.2 Horizontal Pod Autoscaler (HPA) 활용

**HPA 설정 (선택적):**

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: cloud-server-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: cloud-server
  minReplicas: 1
  maxReplicas: 2
  metrics:
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

**주의사항:**
- 4GB 환경에서는 HPA 사용 시 메모리 부족 가능
- 필요시에만 사용

### 8.3 Vertical Pod Autoscaler (VPA) 활용

**VPA 설정 (선택적):**

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: cloud-server-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: cloud-server
  updatePolicy:
    updateMode: "Off"  # 모니터링만 수행
```

**VPA 모드:**
- `Off`: 모니터링만 수행, 자동 조정 안 함
- `Initial`: Pod 생성 시에만 조정
- `Auto`: 자동 조정 (4GB 환경에서는 위험)

---

## 9. Best Practices

### 9.1 메모리 최적화 체크리스트

**애플리케이션 레벨:**
- [ ] JVM 힙 메모리 최적화 (JAVA_OPTS)
- [ ] 메타스페이스 크기 제한
- [ ] GC 설정 최적화 (G1GC 사용)
- [ ] 커넥션 풀 크기 최적화

**인프라 레벨:**
- [ ] 리소스 requests/limits 설정
- [ ] Pod별 메모리 제한 설정
- [ ] OOMKilled 방지
- [ ] 모니터링 및 알림 설정

**서비스 레벨:**
- [ ] MySQL InnoDB 버퍼 풀 크기 최적화
- [ ] Redis maxmemory 설정
- [ ] Elasticsearch 힙 메모리 최적화
- [ ] Kafka 힙 메모리 최적화

### 9.2 점진적 최적화 전략

**1단계: JVM 힙 메모리 최적화**
- JAVA_OPTS 조정
- 메모리 사용량 모니터링

**2단계: 리소스 requests/limits 설정**
- 각 Pod에 메모리 제한 설정
- OOMKilled 방지

**3단계: 서비스별 최적화**
- 각 서비스의 특성에 맞는 설정 조정
- 불필요한 메모리 사용 최소화

**4단계: 서비스 외부화 검토**
- 필요시 관리형 서비스로 이전
- 메모리 부담 감소

---

## 10. 문제 해결

### 10.1 OOMKilled 발생 시

**원인 분석:**
- limits가 너무 작게 설정됨
- JVM 힙 메모리가 limits를 초과
- 메모리 누수 발생

**해결 방법:**
1. limits 증가 (가능한 경우)
2. JVM 힙 메모리 감소
3. 메모리 누수 확인 및 수정

### 10.2 메모리 부족으로 인한 Pod 재시작

**증상:**
- Pod가 자주 재시작됨
- 메모리 사용량이 limits에 근접

**해결 방법:**
1. 메모리 사용량 모니터링
2. 불필요한 서비스 제거 또는 외부화
3. 인스턴스 타입 업그레이드 고려 (t3.large 등)

### 10.3 성능 저하

**원인:**
- 메모리 부족으로 인한 스왑 사용
- GC 빈도 증가

**해결 방법:**
1. 메모리 사용량 모니터링
2. GC 로그 분석
3. 메모리 할당 최적화

---

## 마무리

**핵심 포인트:**

- **JVM 힙 메모리 최적화가 가장 중요합니다.** JAVA_OPTS를 통해 힙 크기를 조정하고, G1GC를 사용하여 메모리 효율을 높일 수 있습니다.
- **리소스 requests/limits를 설정하여 OOMKilled를 방지하고, 각 Pod의 메모리 사용량을 제어할 수 있습니다.**
- **각 서비스의 특성에 맞는 메모리 설정을 통해 전체 메모리 사용량을 최적화할 수 있습니다.**
- **4GB 환경에서는 모든 서비스를 한 노드에 배포하기 어려우므로, 필요시 서비스를 외부화하는 것을 고려해야 합니다.**

t3.medium(4GB) 환경에서 여러 Pod를 운영할 때는 메모리 사용량을 세밀하게 관리해야 합니다. JVM 힙 메모리 최적화, 리소스 requests/limits 설정, 서비스별 최적화를 통해 메모리 부족 문제를 해결할 수 있습니다.

다음 글에서는 Kubernetes의 **Pod Disruption Budget(PDB)**을 통해 계획된 중단 중에도 Pod의 가용성을 보장하는 방법을 정리해볼 예정입니다. 🚀

