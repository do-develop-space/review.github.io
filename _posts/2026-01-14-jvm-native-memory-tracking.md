---
layout: post
title: "JVM Native Memory Tracking(NMT): RSS 메모리 세부 분석과 최적화"
date: 2026-01-14
categories: [programming, jvm, performance]
tags: [JVM, NMT, NativeMemoryTracking, RSS, 메모리최적화, Metaspace, DirectMemory, CompressedClassSpace]
---

# JVM Native Memory Tracking(NMT): RSS 메모리 세부 분석과 최적화

이전 글에서 Public EC2(t3.medium 4GB) 환경에서 여러 Pod를 운영할 때 메모리 최적화 방법을 다뤘습니다. 이번 글에서는 **Native Memory Tracking(NMT)**을 사용하여 JVM의 RSS(Resident Set Size) 내부 메모리를 세부적으로 파악하고, 메타스페이스, 압축 클래스 공간, 다이렉트 메모리 등을 조정하는 방법을 정리해보겠습니다.

JVM의 메모리는 힙 메모리뿐만 아니라 네이티브 메모리(메타스페이스, 스레드 스택, 다이렉트 메모리 등)도 포함합니다. NMT를 사용하면 이러한 네이티브 메모리 사용량을 정확히 파악하고 최적화할 수 있습니다.

---

## 1. Native Memory Tracking(NMT)이란?

### 1.1 NMT의 개념

**Native Memory Tracking (NMT):**
- JVM이 사용하는 **네이티브 메모리(비힙 메모리)를 추적**하는 기능
- Java 8부터 정식 지원
- RSS(Resident Set Size) 내부의 메모리 사용량을 세부적으로 분석 가능

**RSS (Resident Set Size):**
- 프로세스가 실제로 사용하는 물리 메모리 크기
- 힙 메모리 + 네이티브 메모리 포함
- `top`, `ps` 명령어로 확인 가능

**JVM 메모리 구조:**

```
RSS (전체 물리 메모리)
├── 힙 메모리 (Heap)
│   ├── Young Generation (Eden, Survivor)
│   └── Old Generation
├── 메타스페이스 (Metaspace)
├── 압축 클래스 공간 (Compressed Class Space)
├── 다이렉트 메모리 (Direct Memory)
├── 스레드 스택 (Thread Stack)
├── 코드 캐시 (Code Cache)
└── 기타 네이티브 메모리
```

### 1.2 NMT를 사용해야 하는 이유

**문제 상황:**

```
JAVA_OPTS: -Xmx512m -Xms512m
실제 RSS: 800MB

힙 메모리: 512MB
나머지 288MB는 어디서 사용되는가? 🤔
```

**NMT로 해결:**
- 네이티브 메모리 사용량 정확히 파악
- 메모리 누수 감지
- 최적화 포인트 발견

---

## 2. NMT 활성화 및 사용법

### 2.1 NMT 활성화

**JAVA_OPTS 설정:**

```bash
# NMT 활성화 (summary 모드)
-XX:NativeMemoryTracking=summary

# NMT 활성화 (detail 모드 - 더 상세한 정보)
-XX:NativeMemoryTracking=detail
```

**예시:**

```yaml
# Kubernetes Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    spec:
      containers:
      - name: app
        image: my-app:latest
        env:
        - name: JAVA_OPTS
          value: >-
            -Xmx512m
            -Xms512m
            -XX:+UseG1GC
            -XX:NativeMemoryTracking=summary
            -XX:MaxMetaspaceSize=128m
            -XX:CompressedClassSpaceSize=64m
            -XX:MaxDirectMemorySize=32m
```

### 2.2 NMT 리포트 확인

**jcmd 명령어 사용:**

```bash
# 프로세스 ID 확인
jps -l

# NMT baseline 설정 (초기 상태 저장)
jcmd <pid> VM.native_memory baseline

# 시간 경과 후 diff 확인
jcmd <pid> VM.native_memory summary.diff

# 상세 리포트
jcmd <pid> VM.native_memory detail
```

**예시 출력:**

```
Native Memory Tracking:

Total: reserved=1234567KB, committed=987654KB

-                 Java Heap (reserved=524288KB, committed=524288KB)
                            (mmap: reserved=524288KB, committed=524288KB)

-                     Class (reserved=131072KB, committed=65536KB)
                            (classes #12345)
                            (malloc=8192KB #45678)
                            (mmap: reserved=122880KB, committed=57344KB)
                            (  Metadata:   )
                            (    reserved=65536KB, committed=57344KB)
                            (    used=51200KB)
                            (  Class space:)
                            (    reserved=65536KB, committed=8192KB)
                            (      malloc=0KB #0)
                            (      mmap: reserved=65536KB, committed=8192KB)
                            (    used=6144KB)

-                    Thread (reserved=24576KB, committed=12288KB)
                            (thread #12)
                            (stack: reserved=24576KB, committed=12288KB)

-                      Code (reserved=65536KB, committed=20480KB)
                            (malloc=2048KB #1234)
                            (mmap: reserved=63488KB, committed=18432KB)

-                        GC (reserved=8192KB, committed=8192KB)
                            (malloc=6144KB #234)
                            (mmap: reserved=2048KB, committed=2048KB)

-                  Compiler (reserved=0KB, committed=0KB)

-                  Internal (reserved=4096KB, committed=4096KB)
                            (malloc=4096KB #567)

-                    Symbol (reserved=16384KB, committed=16384KB)
                            (malloc=16384KB #89012)

-           Native Method Tracking (reserved=1024KB, committed=1024KB)
                            (malloc=1024KB #123)

-               Shared class space (reserved=12288KB, committed=12288KB)
                            (mmap: reserved=12288KB, committed=12288KB)

-                    Unknown (reserved=2048KB, committed=2048KB)
                            (mmap: reserved=2048KB, committed=2048KB)
```

### 2.3 Kubernetes에서 NMT 사용

**Pod 내부에서 확인:**

```bash
# Pod 접속
kubectl exec -it <pod-name> -- /bin/bash

# jcmd 설치 확인 (JDK에 포함)
which jcmd

# 프로세스 확인
jps -l

# NMT 리포트 확인
jcmd 1 VM.native_memory summary
```

**또는 initContainer로 jcmd 설치:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    spec:
      initContainers:
      - name: jcmd-installer
        image: openjdk:17-jdk-slim
        command: ['sh', '-c', 'cp /usr/lib/jvm/java-17-openjdk-amd64/bin/jcmd /shared/']
        volumeMounts:
        - name: shared-tools
          mountPath: /shared
      containers:
      - name: app
        image: my-app:latest
        volumeMounts:
        - name: shared-tools
          mountPath: /usr/local/bin
```

---

## 3. RSS 내부 메모리 구성 요소

### 3.1 힙 메모리 (Heap)

**설정:**
```bash
-Xmx512m  # 최대 힙 크기
-Xms512m  # 초기 힙 크기
```

**NMT 출력:**
```
Java Heap (reserved=524288KB, committed=524288KB)
```

**특징:**
- 가장 큰 메모리 사용 영역
- `-Xmx`로 최대 크기 제한
- GC에 의해 관리됨

### 3.2 메타스페이스 (Metaspace)

**설정:**
```bash
-XX:MaxMetaspaceSize=128m  # 최대 메타스페이스 크기
```

**NMT 출력:**
```
Class (reserved=131072KB, committed=65536KB)
  Metadata:
    reserved=65536KB, committed=57344KB
    used=51200KB
```

**용도:**
- 클래스 메타데이터 저장
- 메서드 메타데이터
- 상수 풀

**특징:**
- Java 8부터 PermGen 대신 사용
- 동적으로 확장 가능 (기본값: 무제한)
- `MaxMetaspaceSize`로 제한 가능

### 3.3 압축 클래스 공간 (Compressed Class Space)

**설정:**
```bash
-XX:CompressedClassSpaceSize=64m  # 압축 클래스 공간 크기
```

**NMT 출력:**
```
Class (reserved=131072KB, committed=65536KB)
  Class space:
    reserved=65536KB, committed=8192KB
    used=6144KB
```

**용도:**
- 64비트 JVM에서 클래스 포인터 압축
- 클래스 메타데이터의 일부 저장

**특징:**
- `-XX:+UseCompressedClassPointers` 활성화 시 사용
- 기본값: 1GB (Java 8), 3GB (Java 11+)
- `CompressedClassSpaceSize`로 제한 가능

**관계:**
```
메타스페이스 = Metadata + Compressed Class Space
```

### 3.4 다이렉트 메모리 (Direct Memory)

**설정:**
```bash
-XX:MaxDirectMemorySize=32m  # 최대 다이렉트 메모리 크기
```

**NMT 출력:**
```
Internal (reserved=32768KB, committed=32768KB)
```

**용도:**
- NIO (New I/O) 버퍼
- 네트워크 I/O
- 파일 I/O

**특징:**
- 힙 외부의 네이티브 메모리
- GC에 의해 관리되지 않음
- `MaxDirectMemorySize`로 제한 가능
- 기본값: `-Xmx`와 동일

**사용 예시:**
```java
// DirectByteBuffer 사용
ByteBuffer buffer = ByteBuffer.allocateDirect(1024 * 1024);  // 1MB
```

### 3.5 스레드 스택 (Thread Stack)

**설정:**
```bash
-Xss1m  # 스레드당 스택 크기
```

**NMT 출력:**
```
Thread (reserved=24576KB, committed=12288KB)
  (thread #12)
  (stack: reserved=24576KB, committed=12288KB)
```

**계산:**
```
총 스레드 스택 메모리 = 스레드 수 × -Xss
예: 12개 스레드 × 1MB = 12MB
```

**특징:**
- 스레드당 독립적인 스택 공간
- 스레드 수가 많을수록 메모리 사용량 증가
- 기본값: 플랫폼별로 다름 (Linux: 1MB)

### 3.6 코드 캐시 (Code Cache)

**설정:**
```bash
-XX:ReservedCodeCacheSize=48m  # 코드 캐시 예약 크기
-XX:InitialCodeCacheSize=8m    # 초기 코드 캐시 크기
```

**NMT 출력:**
```
Code (reserved=49152KB, committed=20480KB)
```

**용도:**
- JIT 컴파일된 네이티브 코드 저장
- 인터프리터 코드

**특징:**
- JIT 컴파일러가 사용
- 기본값: 240MB (Java 8), 512MB (Java 11+)

---

## 4. 메모리 세부 조정 실전 예시

### 4.1 전체 메모리 구성 예시

**시나리오:**
- 전체 RSS 목표: 512MB
- 힙 메모리: 256MB
- 나머지 네이티브 메모리: 256MB

**JAVA_OPTS 설정:**

```bash
JAVA_OPTS="
  # 힙 메모리
  -Xmx256m
  -Xms256m
  
  # GC 설정
  -XX:+UseG1GC
  -XX:MaxGCPauseMillis=200
  
  # 메타스페이스
  -XX:MaxMetaspaceSize=128m
  
  # 압축 클래스 공간
  -XX:CompressedClassSpaceSize=64m
  
  # 다이렉트 메모리
  -XX:MaxDirectMemorySize=32m
  
  # 스레드 스택
  -Xss512k
  
  # 코드 캐시
  -XX:ReservedCodeCacheSize=32m
  
  # NMT 활성화
  -XX:NativeMemoryTracking=summary
"
```

**예상 메모리 구성:**

```
RSS: ~512MB
├── 힙: 256MB
├── 메타스페이스: 128MB
├── 압축 클래스 공간: 64MB (메타스페이스 내부)
├── 다이렉트 메모리: 32MB
├── 스레드 스택: ~12MB (24개 스레드 × 512KB)
├── 코드 캐시: 32MB
└── 기타: ~10MB
```

### 4.2 NMT로 메모리 확인

**1. 애플리케이션 시작 후 baseline 설정:**

```bash
jcmd <pid> VM.native_memory baseline
```

**2. 시간 경과 후 diff 확인:**

```bash
jcmd <pid> VM.native_memory summary.diff
```

**출력 예시:**

```
Native Memory Tracking:

Total: reserved=536870KB (+123456KB), committed=524288KB (+98765KB)

-                 Java Heap (reserved=262144KB, committed=262144KB)
                            (mmap: reserved=262144KB, committed=262144KB)

-                     Class (reserved=131072KB, committed=65536KB)
                            (classes #12345)
                            (  Metadata:   )
                            (    reserved=65536KB, committed=57344KB)
                            (    used=51200KB)
                            (  Class space:)
                            (    reserved=65536KB, committed=8192KB)
                            (    used=6144KB)

-                    Thread (reserved=12288KB, committed=12288KB)
                            (thread #24)
                            (stack: reserved=12288KB, committed=12288KB)

-                      Code (reserved=32768KB, committed=20480KB)

-                  Internal (reserved=32768KB, committed=32768KB)
                            (malloc=32768KB #123)
```

**3. 메모리 사용량 분석:**

```bash
# 각 영역별 사용량 확인
jcmd <pid> VM.native_memory detail | grep -A 5 "Class\|Thread\|Code\|Internal"
```

### 4.3 메모리 최적화 전략

**1. 메타스페이스 최적화:**

```bash
# 현재 사용량 확인
jcmd <pid> VM.native_memory summary | grep -A 10 "Class"

# 사용량이 80MB인 경우
-XX:MaxMetaspaceSize=128m  # 여유를 두고 설정
```

**2. 압축 클래스 공간 최적화:**

```bash
# 현재 사용량 확인
jcmd <pid> VM.native_memory detail | grep -A 5 "Class space"

# 사용량이 32MB인 경우
-XX:CompressedClassSpaceSize=64m  # 여유를 두고 설정
```

**3. 다이렉트 메모리 최적화:**

```bash
# 현재 사용량 확인
jcmd <pid> VM.native_memory summary | grep "Internal"

# NIO 사용량이 적은 경우
-XX:MaxDirectMemorySize=32m  # 최소한으로 설정
```

**4. 스레드 스택 최적화:**

```bash
# 스레드 수 확인
jcmd <pid> VM.native_memory summary | grep "Thread"

# 스레드 수가 적은 경우
-Xss512k  # 기본값(1MB)보다 작게 설정
```

---

## 5. 실전 시나리오: 4GB 환경 최적화

### 5.1 시나리오 설정

**환경:**
- EC2 t3.medium: 4GB RAM
- 여러 Pod 실행 (control-plane, cloud, frontend, mysql, redis 등)
- 각 Pod당 메모리 제한 필요

**목표:**
- 애플리케이션 Pod: RSS 512MB 이하
- 힙: 256MB
- 네이티브 메모리: 256MB

### 5.2 단계별 최적화

**1단계: 기본 설정으로 시작**

```yaml
env:
- name: JAVA_OPTS
  value: >-
    -Xmx256m
    -Xms256m
    -XX:+UseG1GC
    -XX:NativeMemoryTracking=summary
```

**2단계: NMT로 메모리 분석**

```bash
# Pod 접속
kubectl exec -it <pod-name> -- /bin/bash

# baseline 설정
jcmd 1 VM.native_memory baseline

# 5분 후 diff 확인
jcmd 1 VM.native_memory summary.diff
```

**출력 분석:**
```
Class (reserved=131072KB, committed=98304KB)  # 메타스페이스 사용량 높음
  Metadata: used=90112KB
  Class space: used=8192KB
```

**3단계: 세부 조정**

```yaml
env:
- name: JAVA_OPTS
  value: >-
    -Xmx256m
    -Xms256m
    -XX:+UseG1GC
    -XX:MaxMetaspaceSize=128m
    -XX:CompressedClassSpaceSize=64m
    -XX:MaxDirectMemorySize=32m
    -Xss512k
    -XX:ReservedCodeCacheSize=32m
    -XX:NativeMemoryTracking=summary
```

**4단계: 최종 확인**

```bash
# RSS 확인
kubectl top pod <pod-name>

# NMT로 세부 확인
jcmd 1 VM.native_memory summary
```

### 5.3 최적화 결과

**Before (기본 설정):**
```
RSS: 680MB
├── 힙: 256MB
├── 메타스페이스: 256MB (무제한)
├── 압축 클래스 공간: 128MB (기본값)
├── 다이렉트 메모리: 256MB (기본값 = -Xmx)
├── 스레드 스택: 24MB (24개 × 1MB)
└── 기타: ~60MB
```

**After (세부 조정):**
```
RSS: 480MB
├── 힙: 256MB
├── 메타스페이스: 128MB (제한)
├── 압축 클래스 공간: 64MB (제한)
├── 다이렉트 메모리: 32MB (제한)
├── 스레드 스택: 12MB (24개 × 512KB)
└── 기타: ~28MB
```

**절약: 200MB (29% 감소)**

---

## 6. 메모리 누수 감지

### 6.1 NMT로 메모리 누수 확인

**1. baseline 설정:**

```bash
jcmd <pid> VM.native_memory baseline
```

**2. 시간 경과 후 diff 확인:**

```bash
# 1시간 후
jcmd <pid> VM.native_memory summary.diff
```

**메모리 누수 의심 출력:**

```
Native Memory Tracking:

Total: reserved=536870KB (+123456KB), committed=524288KB (+98765KB)

-                     Class (reserved=196608KB, committed=131072KB)
                            (+65536KB)  # 계속 증가
                            (classes #12345)
                            (  Metadata:   )
                            (    reserved=131072KB, committed=98304KB)
                            (    used=90112KB)  # 계속 증가
```

**문제:**
- 메타스페이스가 계속 증가
- 클래스 로더가 제대로 해제되지 않음

**해결:**
- 클래스 로더 누수 확인
- 동적 클래스 로딩 최소화
- `MaxMetaspaceSize`로 제한

### 6.2 다이렉트 메모리 누수 확인

**NMT 출력:**

```
Internal (reserved=65536KB, committed=65536KB)
  (+32768KB)  # 계속 증가
```

**문제:**
- DirectByteBuffer가 해제되지 않음
- NIO 사용 후 버퍼 해제 누락

**해결:**
```java
// ❌ 버퍼 해제 누락
ByteBuffer buffer = ByteBuffer.allocateDirect(1024 * 1024);

// ✅ 명시적 해제 (또는 try-with-resources)
DirectByteBuffer buffer = (DirectByteBuffer) ByteBuffer.allocateDirect(1024 * 1024);
// 사용 후
((DirectBuffer) buffer).cleaner().clean();
```

---

## 7. Best Practices

### 7.1 메모리 설정 권장사항

**작은 환경 (512MB-1GB):**
```bash
-Xmx256m
-Xms256m
-XX:MaxMetaspaceSize=128m
-XX:CompressedClassSpaceSize=64m
-XX:MaxDirectMemorySize=32m
-Xss512k
-XX:ReservedCodeCacheSize=32m
```

**중간 환경 (1GB-2GB):**
```bash
-Xmx512m
-Xms512m
-XX:MaxMetaspaceSize=256m
-XX:CompressedClassSpaceSize=128m
-XX:MaxDirectMemorySize=64m
-Xss1m
-XX:ReservedCodeCacheSize=64m
```

**큰 환경 (2GB+):**
```bash
-Xmx1g
-Xms1g
-XX:MaxMetaspaceSize=512m
-XX:CompressedClassSpaceSize=256m
-XX:MaxDirectMemorySize=128m
-Xss1m
-XX:ReservedCodeCacheSize=128m
```

### 7.2 모니터링 및 알림

**NMT 리포트 주기적 확인:**

```bash
#!/bin/bash
# check_nmt.sh

PID=$(jps -l | grep MyApp | awk '{print $1}')

if [ -z "$PID" ]; then
    echo "Application not found"
    exit 1
fi

# NMT 리포트 저장
jcmd $PID VM.native_memory summary > /tmp/nmt_$(date +%Y%m%d_%H%M%S).txt

# 메타스페이스 사용량 확인
METASPACE=$(jcmd $PID VM.native_memory summary | grep -A 5 "Class" | grep "committed" | awk '{print $2}')

# 임계값 초과 시 알림
if [ "$METASPACE" -gt 100000 ]; then
    echo "WARNING: Metaspace usage is high: ${METASPACE}KB"
fi
```

**Kubernetes CronJob으로 주기적 확인:**

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nmt-checker
spec:
  schedule: "*/30 * * * *"  # 30분마다
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: nmt-checker
            image: openjdk:17-jdk-slim
            command:
            - /bin/bash
            - -c
            - |
              PID=$(jps -l | grep MyApp | awk '{print $1}')
              jcmd $PID VM.native_memory summary.diff
          restartPolicy: OnFailure
```

### 7.3 메모리 최적화 체크리스트

**설정 전:**
- [ ] NMT 활성화 (`-XX:NativeMemoryTracking=summary`)
- [ ] baseline 설정
- [ ] 실제 메모리 사용량 측정

**설정 중:**
- [ ] 메타스페이스 사용량 확인 및 제한 설정
- [ ] 압축 클래스 공간 사용량 확인 및 제한 설정
- [ ] 다이렉트 메모리 사용량 확인 및 제한 설정
- [ ] 스레드 스택 크기 최적화
- [ ] 코드 캐시 크기 최적화

**설정 후:**
- [ ] RSS 메모리 확인
- [ ] NMT 리포트로 세부 확인
- [ ] 메모리 누수 확인 (diff)
- [ ] 성능 테스트 (GC 로그 확인)

---

## 8. 문제 해결

### 8.1 OutOfMemoryError: Metaspace

**에러:**
```
java.lang.OutOfMemoryError: Metaspace
```

**원인:**
- 메타스페이스가 `MaxMetaspaceSize`를 초과

**해결:**
```bash
# 1. 현재 사용량 확인
jcmd <pid> VM.native_memory summary | grep -A 10 "Class"

# 2. MaxMetaspaceSize 증가
-XX:MaxMetaspaceSize=256m  # 기존 128m에서 증가

# 3. 클래스 로더 누수 확인
jcmd <pid> VM.classloader_stats
```

### 8.2 OutOfMemoryError: Direct buffer memory

**에러:**
```
java.lang.OutOfMemoryError: Direct buffer memory
```

**원인:**
- 다이렉트 메모리가 `MaxDirectMemorySize`를 초과

**해결:**
```bash
# 1. 현재 사용량 확인
jcmd <pid> VM.native_memory summary | grep "Internal"

# 2. MaxDirectMemorySize 증가
-XX:MaxDirectMemorySize=64m  # 기존 32m에서 증가

# 3. DirectByteBuffer 사용 코드 확인
# 버퍼 해제 누락 확인
```

### 8.3 RSS가 예상보다 높은 경우

**원인 분석:**

```bash
# 1. NMT로 전체 메모리 확인
jcmd <pid> VM.native_memory summary

# 2. 각 영역별 사용량 확인
# - 힙 메모리
# - 메타스페이스
# - 스레드 스택
# - 코드 캐시
# - 다이렉트 메모리

# 3. 문제 영역 식별 및 최적화
```

**해결:**
- 각 영역별 제한 설정
- 불필요한 메모리 사용 최소화
- 스레드 수 최적화

---

## 마무리

**핵심 포인트:**

- **Native Memory Tracking(NMT)은 JVM의 네이티브 메모리 사용량을 정확히 파악할 수 있는 강력한 도구입니다.**
- **RSS는 힙 메모리뿐만 아니라 메타스페이스, 압축 클래스 공간, 다이렉트 메모리 등 모든 네이티브 메모리를 포함합니다.**
- **메타스페이스, 압축 클래스 공간, 다이렉트 메모리를 세부 조정하여 전체 RSS 메모리를 최적화할 수 있습니다.**
- **NMT를 사용하여 메모리 누수를 감지하고, 각 영역별 사용량을 모니터링하여 지속적으로 최적화할 수 있습니다.**

**최종 권장사항:**

1. **NMT를 항상 활성화하여 메모리 사용량을 모니터링**
2. **메타스페이스, 압축 클래스 공간, 다이렉트 메모리를 실제 사용량에 맞게 제한 설정**
3. **정기적으로 NMT 리포트를 확인하여 메모리 누수 감지**
4. **제한된 메모리 환경에서는 각 영역별 세부 조정이 필수적**

NMT를 활용하면 JVM의 메모리 사용량을 세부적으로 분석하고 최적화할 수 있어, 제한된 메모리 환경에서도 안정적으로 애플리케이션을 운영할 수 있습니다. 🚀



