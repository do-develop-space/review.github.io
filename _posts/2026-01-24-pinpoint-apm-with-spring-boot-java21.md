---
layout: post
title: "Pinpoint APM으로 Spring Boot 3.x / Java 21 모니터링하기"
date: 2026-01-24
categories: [devops, monitoring, apm]
tags: [Pinpoint, APM, SpringBoot, Java21, 모니터링, 성능, 트레이싱, HBase]
---

이전 글들에서 Kafka의 동기/비동기 성능 차이와 Consumer 처리 방식을 정리했습니다.  
이번 글에서는 **Spring Boot 3.x + Java 21 애플리케이션을 Pinpoint로 모니터링하는 방법**을, *공부하면서 따라가는 느낌*으로 정리해보겠습니다.

글의 많은 내용은 실제 사용기를 바탕으로 한 블로그 글을 참고하여 재구성했습니다.  
특히 버전 조합과 설치 순서는 [`Pinpoint로 APM 구축하기 With SpringBoot`](https://dgjinsu.tistory.com/91)를 많이 참고했습니다.

---

## 1. APM과 Pinpoint를 먼저 이해하자

### 1.1 APM이란?

**APM(Application Performance Management / Monitoring)** 은 말 그대로 **애플리케이션 성능을 관리/모니터링**하는 도구입니다.

관찰 대상:

- CPU, 메모리, 스레드 개수
- 트랜잭션 수, 응답 시간, 에러 비율
- 외부 시스템 호출(DB, Redis, 외부 API 등)

APM 도구를 쓰면 다음과 같은 질문에 답하기 쉬워집니다.

- "지금 서버가 느린데 **CPU 때문인지, DB 때문인지** 알고 싶다."
- "어떤 **요청 URL**이 병목인지 알고 싶다."
- "배포 이후에 **응답 시간이 얼마나 늘었는지** 확인하고 싶다."

### 1.2 Prometheus + Grafana vs Pinpoint

실무에서 많이 쓰는 모니터링 스택은 **Prometheus + Grafana**와 **Pinpoint** 입니다. 둘은 역할이 겹치지만, **초점이 조금 다릅니다**. [`참고`](https://dgjinsu.tistory.com/91)

- **Prometheus + Grafana**
  - **메트릭(metric) 중심**: CPU, JVM 메모리, TPS, 에러율 등
  - "지금 시스템 전체 상태가 어떤지" → **숲을 보는 용도**
  - 그래프를 보고 "이 시간대에 CPU가 90%였구나" 정도를 파악하기 좋음

- **Pinpoint**
  - **트레이싱(tracing) 중심**: 한 HTTP 요청이 어떤 서비스/메서드/쿼리를 거쳤는지 추적
  - "이 요청이 **어디에서 느려졌는지, 어느 서비스가 병목인지**" → **나무를 보는 용도**
  - 타임라인, 호출 트리(Call Tree), 분산 트랜잭션 맵을 제공

한 줄로 요약하면:

> - **Prometheus/Grafana = 숲(전체 리소스 상태)**  
> - **Pinpoint = 나무(특정 요청의 상세 흐름과 병목)**  

둘은 대체 관계가 아니라 **함께 쓰는 조합**이라고 보는 게 좋습니다.

---

## 2. 전체 아키텍처와 구성 요소

Pinpoint는 크게 네 가지 컴포넌트로 이뤄집니다.

1. **Agent**
   - Spring Boot 애플리케이션(JVM)에 붙는 Java Agent
   - 메서드 호출, SQL, 외부 호출 정보를 가로채서 Collector로 보냄
2. **Collector**
   - 여러 Agent에서 날아오는 데이터를 받아서 저장소(HBase 등)에 적재
3. **HBase (또는 Elasticsearch)**
   - Pinpoint가 수집한 메트릭/트레이스 데이터를 저장하는 분산 DB
4. **Web UI**
   - 수집된 데이터를 시각화하는 UI (트랜잭션 맵, Call Tree, 응답 시간 분포 등)

구조를 그림으로 그리면 대략 이렇게 됩니다:

```text
Spring Boot App (Java 21)
   ▲
   │ (Agent)
   │
   ▼
Pinpoint Collector ───► HBase (1.2.x)
   │
   ▼
Pinpoint Web UI
```

- 애플리케이션 서버는 **Agent만 붙이면 되고**,  
- Collector/HBase/Web은 보통 **별도 서버(또는 VM, 컨테이너)** 에 둡니다.

---

## 3. Java 21 / Spring Boot 3.x에서 버전 조합 잡기

먼저 가장 헷갈릴 수 있는 **버전 호환성**부터 정리합니다. [`참고`](https://dgjinsu.tistory.com/91)

### 3.1 왜 버전 조합이 중요할까?

- Pinpoint Agent는 **JVM 바이트코드를 조작하는 라이브러리**이기 때문에  
  **Java 버전**과의 호환성이 매우 중요합니다.
- 예를 들어, 예전 Agent 버전은 Java 21의 바이트코드를 이해하지 못해서
  - 애플리케이션이 부팅되지 않거나
  - 모니터링 데이터가 제대로 수집되지 않을 수 있습니다.

### 3.2 글에서 사용한 조합

예제로 참고한 글에서는 다음과 같은 구성을 사용해 **Spring Boot 3.x + Java 21** 환경을 모니터링했습니다. [`참고`](https://dgjinsu.tistory.com/91)

- **애플리케이션 서버 (Server1)**
  - Spring Boot 3.x
  - **Java 21**
  - Pinpoint **Agent 3.0.1**  (※ 3.0.1 미만 Agent는 Java 21 미지원)

- **APM 서버 (Server2)**
  - Java 8 (HBase, Collector, Web 구동용)
  - HBase **1.2.7**
  - Pinpoint Collector **2.2.2**
  - Pinpoint Web **2.2.2**

핵심만 정리하면:

- **Agent는 3.0.1 이상 (Java 21 지원 버전)**
- **Collector/Web은 2.2.x 사용**
- **HBase는 1.2.x (1.2.7 권장)**
- HBase/Collector/Web은 Java 8로 돌리고,  
  애플리케이션은 Java 21로 돌리는 **2개 Java 버전 공존 구조**를 씁니다.

---

## 4. Server2: HBase + Collector + Web 설치 (APM 서버)

이제부터는 "나도 직접 한 번 구축해보자"는 느낌으로, **APM 서버(Server2)** 에서 할 작업을 순서대로 정리해보겠습니다.

### 4.1 HBase 1.2.7 설치

```bash
wget https://archive.apache.org/dist/hbase/1.2.7/hbase-1.2.7-bin.tar.gz
tar xzvf hbase-1.2.7-bin.tar.gz
```

`conf/hbase-env.sh`에서 예전 Java 옵션(PermGen 관련)을 주석 처리합니다:

```bash
vi ./hbase-1.2.7/conf/hbase-env.sh

# 아래 옵션들을 주석 처리
# export HBASE_MASTER_OPTS="$HBASE_MASTER_OPTS -XX:PermSize=128m -XX:MaxPermSize=128m"
# export HBASE_REGIONSERVER_OPTS="$HBASE_REGIONSERVER_OPTS -XX:PermSize=128m -XX:MaxPermSize=128m"
```

그리고 **Java 8** 환경에서 HBase를 띄웁니다:

```bash
JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64 \
  ./hbase-1.2.7/bin/start-hbase.sh
```

> 여기서 포인트:  
> - 시스템 기본 Java가 21이어도,  
>   `JAVA_HOME=...`을 지정해주면 해당 스크립트만 Java 8로 실행할 수 있습니다.

### 4.2 Pinpoint용 HBase 테이블 생성

Pinpoint에서 제공하는 스크립트로 HBase 테이블들을 한 번에 생성합니다. [`참고`](https://dgjinsu.tistory.com/91)

```bash
# 스크립트 다운로드
wget https://raw.githubusercontent.com/pinpoint-apm/pinpoint/master/hbase/scripts/hbase-create.hbase

# 스크립트 실행 (역시 Java 8 환경에서)
JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64 \
  ./hbase-1.2.7/bin/hbase shell hbase-create.hbase
```

### 4.3 Pinpoint Collector 실행

```bash
# Collector JAR 다운로드
wget https://github.com/pinpoint-apm/pinpoint/releases/download/v2.2.2/pinpoint-collector-boot-2.2.2.jar

chmod +x pinpoint-collector-boot-2.2.2.jar

# 백그라운드 실행
nohup java -jar \
  -Dpinpoint.zookeeper.address=localhost \
  pinpoint-collector-boot-2.2.2.jar \
  > collector.log 2> collector-error.log &
```

- 실제 운영에서는 Zookeeper/HBase 클러스터 주소를 환경에 맞게 바꿔야 합니다.

### 4.4 Pinpoint Web UI 실행

```bash
# Web JAR 다운로드
wget https://github.com/pinpoint-apm/pinpoint/releases/download/v2.2.2/pinpoint-web-boot-2.2.2.jar

chmod +x pinpoint-web-boot-2.2.2.jar

nohup java -jar \
  -Dpinpoint.zookeeper.address=localhost \
  pinpoint-web-boot-2.2.2.jar \
  > web.log 2> web-error.log &
```

이제 브라우저에서 `http://<Server2 IP>:8079` (기본 포트는 버전에 따라 다를 수 있음)를 열면 Pinpoint Web UI에 접근할 수 있습니다.

---

## 5. Server1: Spring Boot 3.x / Java 21 애플리케이션에 Agent 붙이기

이제 실제 **Spring Boot 애플리케이션 쪽(Server1)**에 Pinpoint Agent를 붙여보겠습니다.

### 5.1 Agent 다운로드

Pinpoint 릴리즈 페이지에서 **Agent 3.0.1**을 다운로드합니다. [`참고`](https://dgjinsu.tistory.com/91)

```bash
wget https://github.com/pinpoint-apm/pinpoint/releases/download/v3.0.1/pinpoint-agent-3.0.1.tar.gz
tar xzvf pinpoint-agent-3.0.1.tar.gz
```

생성된 디렉터리(예: `pinpoint-agent-3.0.1/`) 경로를 기억해둡니다.

### 5.2 Agent 설정 (pinpoint.config)

`pinpoint-root.config` 혹은 `pinpoint.config`에서 Collector 주소 등을 설정합니다.

예시:

```properties
profiler.collector.ip=SERVER2_IP_OR_HOSTNAME
profiler.collector.tcp.port=9994
profiler.collector.udp.port=9995
profiler.collector.stat.port=9996

profiler.applicationName=yellow-store
profiler.transport.grpc.collector.ip=SERVER2_IP_OR_HOSTNAME
```

- `profiler.applicationName` 은 Pinpoint UI에서 보이는 **애플리케이션 이름**
- Collector IP/Port는 Server2 설정에 맞게 변경

### 5.3 Spring Boot 실행 시 Agent 붙이기

Jar 실행 시 **JVM 옵션**으로 Agent를 붙입니다:

```bash
java \
  -javaagent:/path/to/pinpoint-agent-3.0.1/pinpoint-bootstrap-3.0.1.jar \
  -Dpinpoint.agentId=yellow-store-1 \
  -Dpinpoint.applicationName=yellow-store \
  -jar yellow-store-0.0.1-SNAPSHOT.jar
```

중요한 포인트:

- **`-javaagent`**: Pinpoint Agent JAR 경로
- **`pinpoint.agentId`**: 인스턴스별로 유니크하게 설정 (예: `yellow-store-1`, `yellow-store-2`)
- **`pinpoint.applicationName`**: Pinpoint UI에 표시될 논리 애플리케이션 이름

Kubernetes 환경이라면 Deployment의 `JAVA_TOOL_OPTIONS`나 `JAVA_OPTS`에 위 옵션을 넣어주면 됩니다.

---

## 6. Pinpoint가 실제로 보여주는 것들

Agent를 붙이고 애플리케이션을 올린 뒤, Pinpoint Web에서 확인할 수 있는 대표 화면들은 다음과 같습니다.

1. **애플리케이션 맵(Application Map)**
   - 어떤 서비스가 어떤 서비스를 호출하는지, 호출량/응답시간/에러 비율을 한눈에 볼 수 있는 지도
2. **트랜잭션 리스트**
   - 특정 시간 동안 들어온 요청 목록, URI, 응답 시간, 에러 여부 등
3. **Call Tree (호출 트리)**
   - 한 요청이 컨트롤러 → 서비스 → 레포지토리 → DB 쿼리 순으로 호출된 흐름과 각 단계의 소요 시간을 시각화
4. **JVM/시스템 메트릭**
   - JVM 메모리, GC, 스레드 수, TPS 등 기본적인 모니터링 정보

개발/운영 관점에서 특히 유용한 시나리오:

- "예약 생성 API가 간헐적으로 2초 이상 걸린다" →  
  Pinpoint Call Tree에서 확인해보면
  - **DB 쿼리 한 개가 1.8초를 잡아먹는다**거나
  - **외부 결제 API 호출이 느리다**는 것을 바로 알 수 있음

---

## 7. 공부하면서 정리한 포인트들

마지막으로, 실제로 Pinpoint를 공부하면서 느낀 **정리 포인트**를 bullet로 남겨둡니다.

- **1) Pinpoint는 Prometheus/Grafana를 대체하는 게 아니라 보완재**
  - 리소스 전체 상태(숲)는 Prometheus/Grafana,  
    특정 요청/트랜잭션(나무)는 Pinpoint로 보는 것이 좋다.

- **2) Java 21을 쓴다면 Agent 버전을 꼭 확인**
  - 3.0.1 미만 Agent는 Java 21 바이트코드를 제대로 다루지 못한다.
  - 애플리케이션이 아예 안 뜨거나, 메트릭이 안 찍힐 수 있다.

- **3) APM 서버 쪽은 Java 8 + HBase 1.2.x 조합이 아직 표준**
  - Pinpoint 문서/사례에서 여전히 이 조합을 많이 사용한다.
  - 운영 환경에서는 Docker Compose나 Kubernetes로 묶어서 올리는 것도 고려.

- **4) Agent 옵션은 코드가 아니라 실행 스크립트(JVM 옵션)에서 관리**
  - 배포 스크립트, Helm Chart, Kubernetes Manifest 등에서 관리하는 게 좋다.
  - 애플리케이션 코드에는 Pinpoint 의존성이 거의 없다.

- **5) 첫 도입 시에는 "한두 개 서비스"부터 시작**
  - MSA 전 서비스에 한 번에 붙이지 말고,  
    트래픽이 많거나 병목이 있는 서비스에 우선 적용해 보는 것이 좋다.

---

## 마무리

이 글에서는 **Spring Boot 3.x / Java 21 환경에서 Pinpoint APM을 도입하는 전체 과정**을 공부하듯 정리했습니다.

- APM 개념과 Pinpoint의 역할
- Prometheus/Grafana와의 차이 (숲 vs 나무)
- Java 21에서 사용할 수 있는 **버전 조합**
- HBase + Collector + Web 설치 순서
- Spring Boot 애플리케이션에 Agent 붙이는 방법

다음에 Pinpoint를 다시 셋업해야 할 때, 이 글이 **“Cheat Sheet”**처럼 도움이 되면 좋겠습니다. 🚀

다음 글에서는 **Kubernetes Pod Affinity와 Pod AntiAffinity**를 비교하고, 다중 서버 환경에서 어떤 전략을 선택해야 하는지 정리해보겠습니다.