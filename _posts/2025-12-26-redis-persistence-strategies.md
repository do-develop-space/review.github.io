---
layout: post
title: "Redis 영속성 전략: RDB, AOF, 혼합 전략 설계"
date: 2025-12-26
categories: [redis]
tags: [Redis, RDB, AOF, 영속성, 백업, 복구, 캐시]
---

# Redis 영속성 전략: RDB, AOF, 혼합 전략 설계

이전 글에서 Kubernetes 환경에서 Redis를 Pod로 운영할지, Docker Compose로 운영할지를 정리했는데, 이번에는 **Redis 자체의 영속성(persistence) 전략**에 대해 살펴보겠습니다.

Redis는 기본적으로 **메모리 기반** 데이터 저장소이지만, 운영 환경에서는 장애 복구, 재시작, 백업을 위해 **디스크에 데이터를 어떻게 남길 것인지**를 반드시 고민해야 합니다.

Redis의 영속성은 크게 세 가지 방식으로 나눌 수 있습니다:

1. **RDB (Snapshot)**
2. **AOF (Append Only File)**
3. **RDB + AOF 혼합 전략**

각 방식의 특징과 설정 방법, 운영 시 고려사항을 정리해보겠습니다.

---

## 1. RDB (Snapshot) 방식

### 개념

**RDB(Redis Database File)**는 **주기적으로 메모리 상태를 스냅샷(snapshot)으로 떠서 디스크에 저장**하는 방식입니다.

- 특정 시점의 메모리 상태를 `.rdb` 파일로 저장
- Redis 재시작 시 이 파일을 읽어 메모리 상태 복원

### 설정 예시

```conf
# redis.conf

# save <seconds> <changes>
# 60초 동안 1000개의 키가 변경되면 snapshot 생성
save 60 1000

# 5분 동안 100개의 키가 변경되면 snapshot 생성
save 300 100

# 15분 동안 1개 이상의 키가 변경되면 snapshot 생성
save 900 1

# RDB 파일 이름
dbfilename dump.rdb

# RDB 파일을 저장할 디렉토리
-dir /data
```

### 장점

- ✅ **디스크 I/O가 상대적으로 적음** (주기적으로만 저장)
- ✅ 파일 크기가 비교적 작고, **백업/복제 용이**
- ✅ 장애 발생 시 **최근 스냅샷 시점까지는 빠르게 복구** 가능

### 단점

- ❌ **스냅샷 이후의 데이터는 유실될 수 있음**
  - 예: 60초마다 snapshot → 마지막 snapshot 이후 59초 동안의 데이터는 장애 시 손실 가능
- ❌ 스냅샷 생성 시점에 **CPU/디스크 부하**가 순간적으로 증가할 수 있음

### 언제 RDB를 쓸까?

- 캐시 용도로 사용하고, **일부 데이터 유실을 허용**할 수 있을 때
- 주기적인 백업/복구가 중요하지만, 실시간 로그 수준의 정확성이 필요 없을 때

---

## 2. AOF (Append Only File) 방식

### 개념

**AOF(Append Only File)**는 **모든 write 명령을 순차적으로 로그 형태로 기록**하는 방식입니다.

- `SET`, `HSET`, `LPUSH` 등 쓰기 명령을 그대로 파일에 append
- Redis 재시작 시 AOF 파일을 순차 실행하여 메모리 상태 복원

### 설정 예시

```conf
# redis.conf

appendonly yes
appendfilename "appendonly.aof"

# fsync 정책
# always: 매 명령마다 fsync (가장 안전하지만 가장 느림)
# everysec: 1초마다 fsync (일반적으로 가장 많이 사용)
# no: OS에 맡김 (가장 빠르지만 안전하지 않음)
appendfsync everysec

# AOF 파일 rewrite 설정
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
```

### 장점

- ✅ **데이터 유실을 최소화**할 수 있음
  - `appendfsync everysec` 기준: 최악의 경우 1초 이내 데이터 유실
- ✅ 운영 중에도 AOF rewrite로 **파일 크기를 정리** 가능
- ✅ RDB보다 **더 세밀한 복구** 가능 (커맨드 단위)

### 단점

- ❌ RDB보다 **디스크 용량을 더 많이 사용**할 수 있음
- ❌ 쓰기 부하가 많은 시스템에서는 **디스크 I/O 부담**이 커질 수 있음
- ❌ 재시작 시 AOF 파일을 처음부터 재실행해야 하므로, **복구 시간이 더 길어질 수 있음**

### 언제 AOF를 쓸까?

- 데이터 유실을 **최소화**해야 하는 경우
- Redis를 단순 캐시가 아니라, **중요한 세션/상태 저장소로 사용하는 경우**

---

## 3. RDB + AOF 혼합 전략

실제 운영에서는 **RDB와 AOF를 함께 사용하는 전략**을 많이 선택합니다.

### 혼합 전략 개념

- RDB: **주기적인 전체 스냅샷**
- AOF: **스냅샷 사이의 변경분을 로그로 기록**

Redis 4.0 이후에는 **AOF 파일을 RDB 스냅샷 + AOF delta 형태**로 운용하는 것도 가능합니다.

### 설정 예시 (둘 다 활성화)

```conf
# RDB 설정
save 900 1
save 300 100
save 60 1000

# AOF 설정
appendonly yes
appendfilename "appendonly.aof"
appendfsync everysec
```

### 장점

- ✅ 재시작 시 **RDB 스냅샷으로 빠르게 복원** 후, AOF로 나머지 변경분 반영
- ✅ RDB 단독 + AOF 단독의 장점을 적절히 섞을 수 있음
- ✅ 백업/복구 전략을 유연하게 가져갈 수 있음

### 단점

- ❌ 설정과 운영이 다소 복잡해짐
- ❌ 디스크 사용량이 증가할 수 있음 (RDB + AOF 모두 저장)

---

## 4. 운영 시 고려사항

### 4.1 장애 허용 범위 정의

Redis 영속성 전략을 정하기 전에 **장애 시 어느 정도의 데이터 유실을 허용할 수 있는지**를 먼저 정의해야 합니다.

- **완전한 무손실**이 필요한가? (사실상 매우 어려움)
- **1초 이내**의 손실은 허용 가능한가? (AOF everysec)
- **수십 초 ~ 수분** 정도의 손실은 허용 가능한가? (RDB snapshot)

### 4.2 백업 전략

- RDB 파일을 주기적으로 **별도 스토리지(S3, NFS 등)에 백업**
- AOF 파일도 주기적으로 압축/백업
- 스냅샷 시점과 애플리케이션 릴리스 시점을 맞추면 **롤백 전략** 수립에 유리

### 4.3 모니터링

- RDB snapshot 시간, 실패 여부 모니터링
- AOF 파일 크기, rewrite 주기, rewrite 소요 시간 모니터링
- 디스크 사용량, IOPS, latency 모니터링

---

## 5. 캐시 vs 저장소 관점에서의 선택

### 단순 캐시로 사용할 때

- 장애 시 Redis를 비워도 애플리케이션이 **원본 DB에서 다시 채울 수 있다면**
- **RDB만 사용**하거나, 심지어 **영속성을 꺼두는 것**도 고려 가능

```conf
# 캐시 용도일 때 (영속성 비활성화 예시)
save ""        # RDB 비활성화
appendonly no   # AOF 비활성화
```

### 상태 저장소로 사용할 때

- 세션, 토큰, 사용자 상태 등 **유실되면 안 되는 데이터**를 담고 있다면
- **AOF (everysec)** 또는 **RDB + AOF 혼합 전략**을 고려

---

## 6. Kubernetes / Docker 환경에서의 주의점

이전 글에서 Redis를 Pod로 운영할지 Docker Compose로 운영할지 고민했는데, 어떤 방식을 택하든 **영속성 전략 + 스토리지 구성**을 함께 봐야 합니다.

- RDB/AOF 파일을 **로컬 디스크에만 두면**, 노드 장애 시 데이터 유실 위험
- Kubernetes에서는 **PersistentVolume (예: EBS, Ceph, NFS)**와 함께 사용
- Docker Compose 환경에서는 **호스트 볼륨 마운트**로 `/data` 디렉토리를 외부 스토리지에 연결

```yaml
# docker-compose 예시
services:
  redis:
    image: redis:7-alpine
    volumes:
      - ./redis-data:/data
    command: ["redis-server", "/usr/local/etc/redis/redis.conf"]
```

---

## 마무리

**핵심 포인트:**

- Redis의 영속성은 **RDB, AOF, 혼합 전략**으로 나눌 수 있으며, **데이터 유실 허용 범위와 성능 요구사항**에 따라 선택해야 합니다.
- 단순 캐시라면 **RDB 또는 영속성 비활성화**도 고려 가능하지만, 세션/상태 저장소라면 **AOF 또는 RDB+AOF**가 더 적합합니다.
- Kubernetes/Docker 환경에서는 **스토리지 구성(PV, 호스트 볼륨)**까지 함께 설계해야 합니다.

다음 글에서는 이러한 저장소와 데이터를 활용하는 방향으로, **Elasticsearch의 vector(임베딩) 필드와 벡터 검색**에 대해 정리해보겠습니다. 🚀
