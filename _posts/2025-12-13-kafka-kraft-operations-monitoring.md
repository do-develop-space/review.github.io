---
layout: post
title: "Kafka KRaft 모드 운영 및 모니터링 전략"
date: 2025-12-13
categories: [kafka]
tags: [Kafka, KRaft, 운영, 모니터링, 클러스터관리, 프로덕션]
---

# Kafka KRaft 모드 운영 및 모니터링 전략

Kafka가 KRaft 모드로 전환되면서, 운영 방식과 모니터링 전략도 크게 변화했습니다.  
ZooKeeper 의존성이 제거되면서 **단일 클러스터 운영**이 가능해졌지만, 동시에 새로운 운영 고려사항들이 생겼습니다.

이 글에서는 KRaft 모드에서의 실제 운영 방법, 모니터링 전략, 그리고 주의해야 할 사항들을 정리해보겠습니다.

---

## 1. KRaft 모드 운영 개요

### KRaft 모드의 운영 특징

KRaft 모드에서는 ZooKeeper 없이 Kafka 자체적으로 메타데이터를 관리합니다:

- **컨트롤러 노드**: 메타데이터를 관리하는 전용 노드 (Controller Node)
- **브로커 노드**: 데이터를 저장하고 처리하는 노드 (Broker Node)
- **단일 클러스터**: ZooKeeper 클러스터가 필요 없어 운영이 단순해짐

### 아키텍처 비교

**기존 (ZooKeeper 모드):**
```
┌─────────────┐     ┌─────────────┐
│   Broker    │◄────┤  ZooKeeper  │
│   Cluster   │     │   Cluster   │
└─────────────┘     └─────────────┘
```

**KRaft 모드:**
```
┌─────────────┐
│  Controller │
│   Nodes     │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│   Broker    │
│   Nodes     │
└─────────────┘
```

---

## 2. KRaft 모드 설정 및 구성

### Controller 노드 설정

Controller 노드는 메타데이터를 관리하는 핵심 노드입니다:

```properties
# server.properties (Controller 노드)
process.roles=controller
controller.quorum.voters=1@controller1:9093,2@controller2:9093,3@controller3:9093
controller.listener.names=CONTROLLER
listeners=CONTROLLER://controller1:9093
advertised.listeners=CONTROLLER://controller1:9093
```

**주요 설정:**
- `process.roles=controller`: 이 노드가 Controller 역할을 수행
- `controller.quorum.voters`: Controller 노드들의 목록 (홀수 개 권장)
- `controller.listener.names`: Controller 전용 리스너 이름

### Broker 노드 설정

Broker 노드는 데이터를 저장하고 처리합니다:

```properties
# server.properties (Broker 노드)
process.roles=broker
controller.quorum.voters=1@controller1:9093,2@controller2:9093,3@controller3:9093
listeners=PLAINTEXT://broker1:9092
advertised.listeners=PLAINTEXT://broker1:9092
```

**주요 설정:**
- `process.roles=broker`: 이 노드가 Broker 역할을 수행
- `controller.quorum.voters`: Controller 노드 목록 (Broker가 Controller와 통신하기 위해 필요)

### Combined 모드 (Controller + Broker)

소규모 환경에서는 하나의 노드가 Controller와 Broker 역할을 동시에 수행할 수 있습니다:

```properties
# server.properties (Combined 모드)
process.roles=controller,broker
controller.quorum.voters=1@node1:9093,2@node2:9093,3@node3:9093
controller.listener.names=CONTROLLER
listeners=CONTROLLER://node1:9093,PLAINTEXT://node1:9092
advertised.listeners=PLAINTEXT://node1:9092
```

**주의사항:**
- Combined 모드는 소규모 환경에만 권장
- 프로덕션 환경에서는 Controller와 Broker를 분리하는 것이 좋음

---

## 3. 모니터링 전략

### Controller 노드 모니터링

Controller 노드는 클러스터의 메타데이터를 관리하므로, 장애 시 전체 클러스터에 영향을 줍니다.

**핵심 메트릭:**

```bash
# Controller 선출 상태 확인
kafka-metadata-quorum --bootstrap-server localhost:9092 describe --status

# Controller 노드 상태 확인
kafka-metadata-quorum --bootstrap-server localhost:9092 describe --replication
```

**모니터링 항목:**
- Controller 선출 상태 (Active Controller)
- Quorum 상태 (정족수 유지 여부)
- 메타데이터 복제 상태
- Controller 노드의 CPU/메모리 사용률

### Broker 노드 모니터링

Broker 노드는 데이터 저장과 처리를 담당합니다.

**핵심 메트릭:**

```bash
# Broker 상태 확인
kafka-broker-api-versions --bootstrap-server localhost:9092

# 토픽 파티션 상태 확인
kafka-topics --bootstrap-server localhost:9092 --describe
```

**모니터링 항목:**
- Broker 연결 상태
- 파티션 리더/팔로워 상태
- 디스크 사용률
- 네트워크 처리량
- Consumer Lag

### JMX 메트릭

KRaft 모드에서도 JMX를 통해 상세한 메트릭을 수집할 수 있습니다:

**Controller 메트릭:**
- `kafka.controller:type=KafkaController,name=ActiveControllerCount`
- `kafka.controller:type=KafkaController,name=OfflinePartitionsCount`
- `kafka.controller:type=KafkaController,name=PreferredReplicaImbalanceCount`

**Broker 메트릭:**
- `kafka.server:type=BrokerTopicMetrics,name=MessagesInPerSec`
- `kafka.server:type=BrokerTopicMetrics,name=BytesInPerSec`
- `kafka.server:type=BrokerTopicMetrics,name=BytesOutPerSec`

### 모니터링 도구 통합

**Prometheus + Grafana:**
- JMX Exporter를 통해 메트릭 수집
- Grafana 대시보드를 통해 시각화

**예시 Prometheus 설정:**
```yaml
scrape_configs:
  - job_name: 'kafka-controller'
    static_configs:
      - targets: ['controller1:9999']
  - job_name: 'kafka-broker'
    static_configs:
      - targets: ['broker1:9999', 'broker2:9999', 'broker3:9999']
```

---

## 4. 운영 시 주의사항

### Controller 노드 고가용성

Controller 노드는 **홀수 개**로 구성하는 것이 권장됩니다:

- **3개**: 소규모 환경 (1개 장애 허용)
- **5개**: 중규모 환경 (2개 장애 허용)
- **7개 이상**: 대규모 환경 (3개 이상 장애 허용)

**정족수(Quorum) 계산:**
```
정족수 = (노드 수 / 2) + 1
예: 3개 노드 → 정족수 2, 5개 노드 → 정족수 3
```

### 메타데이터 백업

KRaft 모드에서는 메타데이터가 Controller 노드에 저장되므로, 정기적인 백업이 필요합니다:

```bash
# 메타데이터 스냅샷 생성
kafka-metadata-quorum --bootstrap-server localhost:9092 snapshot

# 메타데이터 로그 확인
kafka-dump-log --cluster-metadata-decoder /var/lib/kafka/metadata
```

**백업 전략:**
- 일일 자동 백업
- 중요한 변경 전 수동 백업
- 백업 파일의 암호화 및 오프사이트 저장

### 노드 추가/제거

**Controller 노드 추가:**
1. 새 Controller 노드 설정
2. `controller.quorum.voters`에 새 노드 추가
3. 기존 Controller 노드들 재시작
4. 새 Controller 노드 시작

**Broker 노드 추가:**
1. 새 Broker 노드 설정
2. Broker 노드 시작
3. 파티션 재분배 (필요 시)

### 업그레이드 전략

KRaft 모드에서의 업그레이드는 롤링 업그레이드가 가능합니다:

1. **Controller 노드 업그레이드:**
   - 한 번에 하나씩 업그레이드
   - 정족수를 유지하면서 진행

2. **Broker 노드 업그레이드:**
   - Controller와 독립적으로 업그레이드 가능
   - 파티션 리더 이동을 고려

---

## 5. 마이그레이션 고려사항

### ZooKeeper에서 KRaft로 마이그레이션

기존 ZooKeeper 모드에서 KRaft 모드로 전환하는 과정:

**1단계: 준비**
- KRaft 모드 환경 구성
- 메타데이터 백업

**2단계: Dual Write (선택적)**
- 일부 버전에서는 ZooKeeper와 KRaft 양쪽에 메타데이터 기록
- 점진적 전환 가능

**3단계: 전환**
- ZooKeeper 모드 종료
- KRaft 모드로 완전 전환

**주의사항:**
- 마이그레이션 중 다운타임 발생 가능
- 충분한 테스트 환경에서 검증 필요
- 롤백 계획 수립

### 호환성 확인

**클라이언트 호환성:**
- 기존 Kafka 클라이언트는 KRaft 모드와 호환
- 특별한 변경 없이 사용 가능

**도구 호환성:**
- Kafka 관리 도구들이 KRaft 모드를 지원하는지 확인
- 일부 오래된 도구는 업데이트 필요

---

## 6. 실제 운영 사례

### 성능 개선 사례

KRaft 모드로 전환 후 확인된 개선사항:

- **메타데이터 처리 속도**: 2~3배 향상
- **컨트롤러 선출 시간**: 수십 초 → 수 초로 단축
- **파티션 수 제한**: 수십만 개 → 수백만 개로 증가

### 운영 단순화

**Before (ZooKeeper 모드):**
- Kafka 클러스터 + ZooKeeper 클러스터 운영
- 두 시스템의 모니터링 및 관리 필요
- 복잡한 장애 대응

**After (KRaft 모드):**
- Kafka 클러스터만 운영
- 단일 시스템 모니터링
- 단순한 장애 대응

---

## 7. 모니터링 체크리스트

### 일일 모니터링

- [ ] Controller 노드 상태 확인
- [ ] Broker 노드 상태 확인
- [ ] Consumer Lag 확인
- [ ] 디스크 사용률 확인
- [ ] 네트워크 처리량 확인

### 주간 모니터링

- [ ] 메타데이터 백업 확인
- [ ] 파티션 리더 분산 상태 확인
- [ ] 토픽별 메시지 처리량 추이 분석
- [ ] 에러 로그 분석

### 월간 모니터링

- [ ] 클러스터 성능 분석
- [ ] 용량 계획 수립
- [ ] 보안 설정 점검
- [ ] 업그레이드 계획 검토

---

## 마무리

**핵심 포인트:**

- **Controller 노드 고가용성**: 홀수 개 구성, 정족수 유지
- **모니터링 전략**: Controller와 Broker를 분리하여 모니터링
- **메타데이터 백업**: 정기적인 백업으로 데이터 손실 방지
- **롤링 업그레이드**: 다운타임 없이 업그레이드 가능
- **운영 단순화**: ZooKeeper 제거로 관리 복잡도 감소

KRaft 모드는 Kafka의 운영을 단순화하고 성능을 향상시켰습니다. 하지만 새로운 아키텍처에 맞는 모니터링과 운영 전략이 필요합니다. Controller 노드의 안정성과 메타데이터 관리에 특히 주의를 기울여야 합니다. 🚀

Docker 환경에서 Kafka를 운영할 때는 **프로세스 관리**도 중요한 고려사항입니다. 다음 글에서는 healthcheck 설정으로 인한 zombie 프로세스 문제와 해결 방법에 대해 정리해보겠습니다.

