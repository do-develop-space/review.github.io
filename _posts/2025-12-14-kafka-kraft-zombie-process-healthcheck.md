---
layout: post
title: "Kafka KRaft 모드에서 Healthcheck로 인한 Zombie 프로세스 문제와 해결"
date: 2025-12-14
categories: [kafka, troubleshooting]
tags: [Kafka, KRaft, Healthcheck, Zombie프로세스, Docker, 프로세스관리]
---

# Kafka KRaft 모드에서 Healthcheck로 인한 Zombie 프로세스 문제와 해결

Kafka를 Docker 컨테이너로 운영할 때, healthcheck를 설정하여 컨테이너의 상태를 모니터링하는 경우가 많습니다.  
하지만 잘못된 healthcheck 설정으로 인해 **zombie 프로세스**가 발생하고, 시간이 지날수록 누적되는 문제가 발생할 수 있습니다.

이 글에서는 Kafka KRaft 모드에서 healthcheck로 인한 zombie 프로세스 문제의 원인과 해결 방법을 정리해보겠습니다.

---

## 1. 문제 상황

### 발생하는 현상

Kafka 컨테이너에서 healthcheck를 설정한 후, 다음과 같은 문제가 발생합니다:

- **Zombie 프로세스 누적**: 시간이 지날수록 zombie 프로세스가 계속 증가
- **리소스 누수**: 프로세스 테이블이 점점 차오름
- **성능 저하**: 프로세스 관리 오버헤드 증가

**확인 방법:**
```bash
# 컨테이너 내부에서 zombie 프로세스 확인
docker exec -it kafka-container ps aux | grep defunct

# 또는 호스트에서 확인
docker exec -it kafka-container ps -eo pid,ppid,stat,comm | grep Z
```

### 문제가 되는 Healthcheck 설정

```yaml
healthcheck:
  test: ["CMD-SHELL", "kafka-broker-api-versions --bootstrap-server localhost:9092 || exit 1"]
  interval: 10s  # 10초마다 실행
  timeout: 5s
  retries: 3
```

이 설정은 겉보기에는 정상적으로 보이지만, 실제로는 zombie 프로세스를 생성합니다.

---

## 2. 문제 원인 분석

### Zombie 프로세스란?

**Zombie 프로세스**는 실행이 종료되었지만, 부모 프로세스가 `wait()` 시스템 콜을 호출하지 않아서 프로세스 테이블에 남아있는 프로세스입니다.

- 프로세스는 종료되었지만 프로세스 테이블 엔트리는 남아있음
- 실제 메모리나 CPU를 사용하지 않음
- 하지만 프로세스 테이블 공간을 차지함

### Healthcheck 동작 과정

**문제가 되는 동작 흐름:**

```
1. Docker가 10초마다 healthcheck 실행
   ↓
2. CMD-SHELL이 shell을 통해 명령 실행
   kafka-broker-api-versions --bootstrap-server localhost:9092
   ↓
3. Java 프로세스 생성 (kafka-broker-api-versions는 Java 프로그램)
   ↓
4. 명령 실행 완료 후 프로세스 종료
   ↓
5. Kafka 메인 프로세스가 wait()를 호출하지 않음
   ↓
6. Zombie 프로세스로 남음
```

### 왜 wait()를 호출하지 않는가?

**Kafka의 프로세스 구조:**

```
PID 1 (Kafka 메인 프로세스)
  └─ Java 프로세스 (Kafka Broker)
```

Kafka 메인 프로세스는:
- 자신의 자식 프로세스(Java 프로세스)만 관리
- Healthcheck로 생성된 별도의 Java 프로세스는 자식이 아님
- 따라서 `wait()`를 호출하지 않음

**CMD-SHELL의 문제:**
- `CMD-SHELL`은 shell을 통해 명령을 실행
- Shell이 Java 프로세스를 생성하지만, Kafka 메인 프로세스와의 관계가 명확하지 않음
- 프로세스 정리가 제대로 이루어지지 않음

---

## 3. 해결 방법

### 방법 1: Healthcheck 간격 늘리기 (권장)

가장 간단한 해결 방법은 healthcheck 실행 간격을 늘리는 것입니다:

```yaml
healthcheck:
  test: ["CMD-SHELL", "kafka-broker-api-versions --bootstrap-server localhost:9092 || exit 1"]
  interval: 30s  # 10s → 30s로 변경
  timeout: 5s
  retries: 3
```

**장점:**
- 설정 변경만으로 해결 가능
- Zombie 프로세스 생성 빈도 감소

**단점:**
- 문제 감지 시간이 늘어남 (최대 30초 지연)

**권장 사항:**
- 프로덕션 환경: 30초 이상
- 개발 환경: 60초 이상

### 방법 2: Healthcheck 방식 변경 (CMD 사용)

`CMD-SHELL` 대신 `CMD`를 사용하여 shell 없이 직접 실행:

```yaml
healthcheck:
  test: ["CMD", "kafka-broker-api-versions", "--bootstrap-server", "localhost:9092"]
  interval: 10s
  timeout: 5s
  retries: 3
```

**CMD vs CMD-SHELL 차이:**

**CMD-SHELL:**
```yaml
test: ["CMD-SHELL", "kafka-broker-api-versions --bootstrap-server localhost:9092 || exit 1"]
```
- Shell(`/bin/sh -c`)을 통해 명령 실행
- Shell 연산자 사용 가능 (`||`, `&&`, `;` 등)
- 환경 변수 확장 가능 (`$VAR`)
- Shell 프로세스가 추가로 생성됨
- 프로세스 트리: `PID 1 → shell → Java 프로세스`

**CMD:**
```yaml
test: ["CMD", "kafka-broker-api-versions", "--bootstrap-server", "localhost:9092"]
```
- Shell 없이 직접 실행 (exec 형식)
- 첫 번째 요소가 실행 파일, 나머지는 인자
- Shell 연산자 사용 불가
- 환경 변수 확장 불가
- Shell 프로세스 없이 직접 실행
- 프로세스 트리: `PID 1 → Java 프로세스`

**실제 동작 비교:**

```bash
# CMD-SHELL의 경우
/bin/sh -c "kafka-broker-api-versions --bootstrap-server localhost:9092 || exit 1"
# → shell 프로세스 생성 → shell이 Java 프로세스 실행

# CMD의 경우
kafka-broker-api-versions --bootstrap-server localhost:9092
# → 직접 Java 프로세스 실행 (shell 없음)
```

**장점:**
- Shell 오버헤드 제거 (프로세스 하나 적게 생성)
- 프로세스 관리가 더 명확함
- Zombie 프로세스 발생 가능성 감소

**주의사항:**
- `|| exit 1` 같은 shell 연산자는 사용 불가
- 명령이 실패하면 자동으로 exit code 1 반환 (Docker가 처리)
- 환경 변수를 사용해야 하면 CMD-SHELL 사용 필요

### 방법 3: init 프로세스 사용 (가장 권장)

**init 프로세스란?**

- **init**: Linux 시스템에서 PID 1로 실행되는 첫 번째 프로세스. 모든 프로세스의 부모 프로세스이며, 종료된 자식 프로세스를 정리하는 역할을 합니다.
- **tini (Tiny Init)**: Docker에서 사용하는 경량 init 프로세스입니다. Docker의 `init: true` 옵션은 내부적으로 **tini**를 사용합니다.
- **dumb-init**: 또 다른 경량 init 프로세스로, 수동으로 설치하여 사용할 수 있습니다.

**왜 필요한가?**

일반 Linux 시스템에서는 `systemd`나 `init`이 PID 1로 실행되어 zombie 프로세스를 자동으로 정리합니다. 하지만 Docker 컨테이너에서는 애플리케이션이 PID 1로 실행되는 경우가 많아서, zombie 프로세스 정리 기능이 없습니다.

**해결 방법:**

**옵션 A: Docker의 `init: true` 사용 (가장 간단, 권장)**

```yaml
# docker-compose.yml
services:
  kafka:
    image: confluentinc/cp-kafka:latest
    init: true  # Docker가 자동으로 tini를 사용
    healthcheck:
      test: ["CMD-SHELL", "kafka-broker-api-versions --bootstrap-server localhost:9092 || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 3
```

**옵션 B: Dockerfile에서 tini 직접 사용**

```dockerfile
# Dockerfile
FROM confluentinc/cp-kafka:latest

# tini 설치 (대부분의 이미지에 이미 포함되어 있음)
# RUN apt-get update && apt-get install -y tini

# ENTRYPOINT에 tini 추가
ENTRYPOINT ["tini", "--"]
CMD ["/path/to/kafka-start"]
```

**옵션 C: dumb-init 사용**

```dockerfile
# Dockerfile
FROM confluentinc/cp-kafka:latest

# dumb-init 설치
RUN apt-get update && apt-get install -y dumb-init

# ENTRYPOINT에 dumb-init 추가
ENTRYPOINT ["dumb-init", "--"]
CMD ["/path/to/kafka-start"]
```

**init 프로세스의 역할:**
- PID 1로 실행되어 모든 자식 프로세스 관리
- 종료된 자식 프로세스에 대해 자동으로 `wait()` 호출
- Zombie 프로세스 자동 정리
- 시그널 전파 (SIGTERM, SIGINT 등)

**장점:**
- 가장 확실한 해결 방법
- 다른 프로세스 관리 문제도 함께 해결
- Docker의 `init: true`는 별도 설치 없이 사용 가능

**Docker의 `init: true` 옵션:**
- Docker 1.13+에서 지원
- 내부적으로 **tini**를 사용
- 별도 설치나 설정 없이 간단하게 사용 가능
- 가장 권장되는 방법

### 방법 4: Healthcheck 비활성화 (개발 환경)

개발 환경에서는 healthcheck를 비활성화할 수도 있습니다:

```yaml
services:
  kafka:
    image: confluentinc/cp-kafka:latest
    # healthcheck 주석 처리
    # healthcheck:
    #   test: ["CMD-SHELL", "kafka-broker-api-versions --bootstrap-server localhost:9092 || exit 1"]
    #   interval: 10s
```

**주의사항:**
- 프로덕션 환경에서는 권장하지 않음
- 컨테이너 상태 모니터링이 어려워짐

### 방법 5: 커스텀 Healthcheck 스크립트

프로세스 정리를 포함한 커스텀 스크립트 사용:

```bash
#!/bin/bash
# healthcheck.sh

# Healthcheck 실행
kafka-broker-api-versions --bootstrap-server localhost:9092

# 종료 코드 저장
exit_code=$?

# Zombie 프로세스 정리 (선택적)
# 주의: 이 방법은 근본적인 해결책은 아님
# wait 프로세스가 있으면 정리
wait

exit $exit_code
```

```yaml
healthcheck:
  test: ["CMD-SHELL", "/path/to/healthcheck.sh"]
  interval: 10s
```

**주의사항:**
- 이 방법은 완전한 해결책이 아님
- init 프로세스 사용이 더 권장됨

---

## 4. 권장 설정

### 프로덕션 환경

```yaml
services:
  kafka:
    image: confluentinc/cp-kafka:latest
    init: true  # init 프로세스 사용
    healthcheck:
      test: ["CMD", "kafka-broker-api-versions", "--bootstrap-server", "localhost:9092"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 60s  # 시작 후 60초간 실패 허용
```

**설정 포인트:**
- `init: true`: Zombie 프로세스 자동 정리
- `CMD` 사용: Shell 오버헤드 제거
- `interval: 30s`: 적절한 체크 간격
- `start_period`: 시작 시간 고려

### 개발 환경

```yaml
services:
  kafka:
    image: confluentinc/cp-kafka:latest
    init: true
    healthcheck:
      test: ["CMD", "kafka-broker-api-versions", "--bootstrap-server", "localhost:9092"]
      interval: 60s  # 개발 환경은 더 긴 간격
      timeout: 5s
      retries: 2
```

---

## 5. 모니터링 및 확인

### Zombie 프로세스 모니터링

**정기적인 확인:**
```bash
# 컨테이너 내부에서 확인
docker exec -it kafka-container ps -eo pid,ppid,stat,comm | grep -c " Z "

# 또는 호스트에서
docker exec kafka-container ps aux | grep -c " defunct "
```

**알림 설정:**
- Zombie 프로세스가 일정 개수 이상이면 알림
- Prometheus/Grafana로 모니터링

### Healthcheck 상태 확인

```bash
# 컨테이너 healthcheck 상태 확인
docker ps --format "table {{.Names}}\t{{.Status}}"

# 상세 정보
docker inspect kafka-container | jq '.[0].State.Health'
```

---

## 6. 추가 고려사항

### KRaft 모드에서의 특별한 주의사항

**Controller 노드:**
- Controller 노드도 동일한 문제 발생 가능
- Controller 노드의 healthcheck도 동일하게 설정

**Broker 노드:**
- Broker 노드의 healthcheck는 `kafka-broker-api-versions` 사용
- Controller 노드와 동일한 설정 적용

### 다른 Healthcheck 명령어

**kafka-broker-api-versions 대신 사용 가능한 명령어:**

```yaml
# 방법 1: kafka-broker-api-versions (권장)
test: ["CMD", "kafka-broker-api-versions", "--bootstrap-server", "localhost:9092"]

# 방법 2: kafka-topics (읽기 전용)
test: ["CMD", "kafka-topics", "--bootstrap-server", "localhost:9092", "--list"]

# 방법 3: TCP 연결 확인 (가장 가벼움)
test: ["CMD-SHELL", "nc -z localhost 9092 || exit 1"]
```

**권장:**
- `kafka-broker-api-versions`: Kafka의 실제 상태를 정확히 확인
- TCP 연결: 빠르지만 Kafka 상태를 정확히 반영하지 못할 수 있음

---

## 7. 문제 해결 체크리스트

### 문제 발생 시 확인 사항

- [ ] Healthcheck 설정 확인 (CMD vs CMD-SHELL)
- [ ] Healthcheck 간격 확인 (너무 짧지 않은지)
- [ ] Init 프로세스 사용 여부 확인
- [ ] Zombie 프로세스 개수 확인
- [ ] Kafka 로그 확인 (에러 여부)

### 해결 단계

1. **즉시 조치:**
   - Healthcheck 간격 늘리기 (30초 이상)
   - Init 프로세스 추가 (`init: true`)

2. **근본 해결:**
   - Dockerfile에 init 프로세스 포함
   - Healthcheck 방식을 CMD로 변경

3. **모니터링:**
   - Zombie 프로세스 모니터링 설정
   - 정기적인 확인

---

## 마무리

**핵심 포인트:**

- **문제 원인**: Healthcheck로 생성된 Java 프로세스가 정리되지 않아 zombie 프로세스 발생
- **근본 원인**: Kafka 메인 프로세스가 healthcheck 프로세스에 대해 `wait()`를 호출하지 않음
- **해결 방법**: Init 프로세스 사용 (`init: true`)이 가장 확실한 해결책
- **권장 설정**: Init 프로세스 + CMD 방식 + 적절한 간격(30초 이상)

Kafka KRaft 모드를 Docker로 운영할 때는 **프로세스 관리**에 특히 주의해야 합니다. Healthcheck는 컨테이너 상태 모니터링에 필수적이지만, 잘못된 설정은 오히려 문제를 일으킬 수 있습니다. Init 프로세스를 사용하고 적절한 healthcheck 설정을 통해 안정적인 운영을 유지할 수 있습니다. 🛡️

프로세스 관리와 마찬가지로, **코드 레벨에서의 null 안전성**도 중요합니다. 다음 글에서는 JPA에서 Optional을 올바르게 사용하는 방법과 잘못 사용했을 때 발생하는 문제들에 대해 정리해보겠습니다.

