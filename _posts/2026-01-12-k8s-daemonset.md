---
layout: post
title: "Kubernetes DaemonSet: 모든 노드에 Pod 배포하기"
date: 2026-01-12
categories: [kubernetes, devops]
tags: [Kubernetes, DaemonSet, 데몬셋, 노드별배포, 로깅, 모니터링, 네트워크플러그인]
---

# Kubernetes DaemonSet: 모든 노드에 Pod 배포하기

이전 글에서 Kafka `acks` 옵션을 통해 프로듀서의 메시지 전송 안정성을 보장하는 방법을 다뤘습니다. 이번 글에서는 **DaemonSet**을 통해 클러스터의 모든 노드(또는 특정 노드)에 Pod를 자동으로 배포하는 방법을 정리해보겠습니다.

DaemonSet은 로깅, 모니터링, 네트워크 플러그인 등 **모든 노드에서 실행되어야 하는 시스템 레벨 작업**에 필수적인 Kubernetes 리소스입니다.

---

## 1. DaemonSet이란?

### 1.1 DaemonSet의 개념

**DaemonSet:**
- 클러스터의 **모든 노드(또는 특정 노드)에 정확히 1개의 Pod를 배포**하는 Kubernetes 리소스
- 노드가 추가되면 자동으로 Pod 생성
- 노드가 제거되면 해당 Pod도 자동으로 삭제
- Deployment/ReplicaSet과 달리 **노드 수에 따라 Pod 수가 결정됨**

**핵심 특징:**
- **노드별 1개 Pod**: 각 노드에 정확히 1개의 Pod만 실행
- **자동 확장**: 새 노드 추가 시 자동으로 Pod 생성
- **자동 정리**: 노드 제거 시 Pod 자동 삭제
- **노드 선택**: `nodeSelector`, `nodeAffinity`, `tolerations`로 특정 노드에만 배포 가능

### 1.2 DaemonSet vs Deployment/ReplicaSet

**Deployment/ReplicaSet:**
- **Pod 수 기반**: `replicas: 5` → 5개 Pod 생성
- **노드 무관**: 어떤 노드에 배포될지 스케줄러가 결정
- **사용 사례**: 웹 애플리케이션, API 서버, 마이크로서비스

**DaemonSet:**
- **노드 수 기반**: 노드 3개 → 3개 Pod 생성 (자동)
- **노드별 1개**: 각 노드에 정확히 1개
- **사용 사례**: 로깅 에이전트, 모니터링 에이전트, 네트워크 플러그인

**비교표:**

| 구분 | Deployment/ReplicaSet | DaemonSet |
|------|----------------------|-----------|
| **Pod 수 결정** | `replicas` 설정 | 노드 수 (자동) |
| **노드별 Pod 수** | 여러 개 가능 | 정확히 1개 |
| **노드 추가 시** | 변화 없음 | 자동으로 Pod 생성 |
| **노드 제거 시** | 다른 노드로 재스케줄링 | Pod 삭제 |
| **사용 사례** | 애플리케이션 | 시스템 레벨 작업 |

---

## 2. DaemonSet 기본 사용법

### 2.1 기본 DaemonSet 예시

**모든 노드에 배포:**

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-logging
  namespace: kube-system
spec:
  selector:
    matchLabels:
      name: fluentd-logging
  template:
    metadata:
      labels:
        name: fluentd-logging
    spec:
      containers:
      - name: fluentd
        image: fluent/fluentd:latest
        resources:
          limits:
            memory: 200Mi
          requests:
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

**DaemonSet 생성 및 확인:**

```bash
# DaemonSet 생성
kubectl apply -f daemonset.yaml

# DaemonSet 확인
kubectl get daemonset -n kube-system

# Pod 확인 (노드별 1개씩)
kubectl get pods -n kube-system -l name=fluentd-logging

# 특정 노드의 Pod 확인
kubectl get pods -n kube-system -l name=fluentd-logging -o wide
```

**출력 예시:**

```
NAME              DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
fluentd-logging   3         3         3       3            3           <none>          5m
```

- **DESIRED**: 원하는 Pod 수 (노드 수와 동일)
- **CURRENT**: 현재 생성된 Pod 수
- **READY**: 준비된 Pod 수
- **UP-TO-DATE**: 최신 버전 Pod 수
- **AVAILABLE**: 사용 가능한 Pod 수

### 2.2 특정 노드에만 배포

**nodeSelector 사용:**

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: monitoring-agent
spec:
  selector:
    matchLabels:
      name: monitoring-agent
  template:
    metadata:
      labels:
        name: monitoring-agent
    spec:
      nodeSelector:
        monitoring: "enabled"  # monitoring=enabled 라벨이 있는 노드에만 배포
      containers:
      - name: monitoring
        image: prom/node-exporter:latest
```

**nodeAffinity 사용:**

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: gpu-monitor
spec:
  selector:
    matchLabels:
      name: gpu-monitor
  template:
    metadata:
      labels:
        name: gpu-monitor
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: accelerator
                operator: In
                values:
                - gpu
      containers:
      - name: gpu-monitor
        image: nvidia/dcgm-exporter:latest
```

**tolerations 사용 (Master Node 포함):**

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: cluster-monitor
spec:
  selector:
    matchLabels:
      name: cluster-monitor
  template:
    metadata:
      labels:
        name: cluster-monitor
    spec:
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      containers:
      - name: monitor
        image: prom/node-exporter:latest
```

---

## 3. DaemonSet 사용 사례

### 3.1 로깅 에이전트

**Fluentd, Filebeat 등:**

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: kube-system
spec:
  selector:
    matchLabels:
      name: fluentd
  template:
    metadata:
      labels:
        name: fluentd
    spec:
      containers:
      - name: fluentd
        image: fluent/fluentd-kubernetes-daemonset:v1-debian-elasticsearch
        env:
        - name: FLUENT_ELASTICSEARCH_HOST
          value: "elasticsearch.logging.svc.cluster.local"
        - name: FLUENT_ELASTICSEARCH_PORT
          value: "9200"
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

**특징:**
- 모든 노드의 로그를 수집
- 호스트의 `/var/log` 마운트
- 중앙 로그 시스템(Elasticsearch 등)으로 전송

### 3.2 모니터링 에이전트

**Prometheus Node Exporter:**

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
spec:
  selector:
    matchLabels:
      name: node-exporter
  template:
    metadata:
      labels:
        name: node-exporter
    spec:
      hostNetwork: true
      hostPID: true
      containers:
      - name: node-exporter
        image: prom/node-exporter:latest
        args:
        - --path.procfs=/host/proc
        - --path.sysfs=/host/sys
        - --collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)
        volumeMounts:
        - name: proc
          mountPath: /host/proc
          readOnly: true
        - name: sys
          mountPath: /host/sys
          readOnly: true
      volumes:
      - name: proc
        hostPath:
          path: /proc
      - name: sys
        hostPath:
          path: /sys
```

**특징:**
- 노드 메트릭 수집 (CPU, 메모리, 디스크 등)
- `hostNetwork: true`로 노드 네트워크 직접 접근
- `hostPID: true`로 호스트 프로세스 정보 접근

### 3.3 네트워크 플러그인

**CNI 플러그인 (예: Calico, Flannel):**

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: calico-node
  namespace: kube-system
spec:
  selector:
    matchLabels:
      k8s-app: calico-node
  template:
    metadata:
      labels:
        k8s-app: calico-node
    spec:
      hostNetwork: true
      tolerations:
      - effect: NoSchedule
        operator: Exists
      containers:
      - name: calico-node
        image: calico/node:latest
        env:
        - name: DATASTORE_TYPE
          value: "kubernetes"
        securityContext:
          privileged: true
        volumeMounts:
        - mountPath: /var/run/calico
          name: var-run-calico
        - mountPath: /var/lib/calico
          name: var-lib-calico
      volumes:
      - name: var-run-calico
        hostPath:
          path: /var/run/calico
      - name: var-lib-calico
        hostPath:
          path: /var/lib/calico
```

**특징:**
- 네트워크 오버레이 구성
- `privileged: true`로 호스트 네트워크 스택 접근
- 모든 노드에서 네트워크 정책 적용

### 3.4 보안 스캐너

**보안 취약점 스캐닝:**

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: security-scanner
  namespace: kube-system
spec:
  selector:
    matchLabels:
      name: security-scanner
  template:
    metadata:
      labels:
        name: security-scanner
    spec:
      containers:
      - name: scanner
        image: aquasec/trivy:latest
        securityContext:
          privileged: true
        volumeMounts:
        - name: docker-sock
          mountPath: /var/run/docker.sock
      volumes:
      - name: docker-sock
        hostPath:
          path: /var/run/docker.sock
```

---

## 4. DaemonSet 업데이트 전략

### 4.1 Rolling Update

**DaemonSet 기본 업데이트 방식:**

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1  # 최대 1개 Pod만 동시에 업데이트
  # ... template ...
```

**Rolling Update 동작:**
1. 노드 1의 Pod 삭제 → 새 Pod 생성
2. 노드 2의 Pod 삭제 → 새 Pod 생성
3. 반복...

**특징:**
- 무중단 업데이트
- `maxUnavailable`로 동시 업데이트 수 제한
- 기본값: `maxUnavailable: 1`

### 4.2 OnDelete 전략

**수동 삭제 시에만 업데이트:**

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: critical-daemon
spec:
  updateStrategy:
    type: OnDelete  # Pod를 수동으로 삭제해야 업데이트
  # ... template ...
```

**사용 사례:**
- 중요한 시스템 데몬
- 업데이트 시점을 직접 제어하고 싶을 때

**업데이트 방법:**

```bash
# Pod 수동 삭제
kubectl delete pod <pod-name> -n <namespace>

# DaemonSet이 자동으로 새 Pod 생성 (새 이미지로)
```

---

## 5. DaemonSet과 노드 관리

### 5.1 노드 추가 시

**자동 Pod 생성:**

```
노드 추가
  ↓
DaemonSet Controller 감지
  ↓
새 노드에 Pod 자동 생성
  ↓
Pod Ready 상태 확인
```

**예시:**

```bash
# 노드 추가
kubectl label node new-worker node-role.kubernetes.io/worker=

# DaemonSet이 자동으로 새 노드에 Pod 생성
kubectl get pods -n kube-system -l name=fluentd-logging -o wide
```

### 5.2 노드 제거 시

**자동 Pod 삭제:**

```
노드 제거 시작
  ↓
DaemonSet Controller 감지
  ↓
해당 노드의 Pod 자동 삭제
  ↓
노드 제거 완료
```

**노드 드레이닝:**

```bash
# 노드 드레이닝 (DaemonSet Pod는 --ignore-daemonsets로 유지)
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# 노드 제거 후 DaemonSet Pod 자동 삭제됨
```

### 5.3 노드 유지보수

**노드 유지보수 모드:**

```bash
# 노드에 유지보수 라벨 추가
kubectl label node <node-name> maintenance=true

# DaemonSet에서 해당 노드 제외
# (nodeSelector 또는 nodeAffinity로 제어)
```

**DaemonSet 수정:**

```yaml
spec:
  template:
    spec:
      nodeSelector:
        maintenance: "false"  # 유지보수 노드 제외
```

---

## 6. DaemonSet과 Resource Quota

### 6.1 Resource Quota 적용

**네임스페이스별 Resource Quota:**

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: daemonset-quota
  namespace: kube-system
spec:
  hard:
    pods: "50"  # DaemonSet Pod 포함
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
```

**주의사항:**
- DaemonSet Pod도 Resource Quota에 포함
- 노드 수가 많으면 DaemonSet Pod 수가 증가하여 Quota 초과 가능

### 6.2 LimitRange 적용

**Pod별 리소스 제한:**

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: daemonset-limits
  namespace: kube-system
spec:
  limits:
  - default:
      cpu: "100m"
      memory: "128Mi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
    type: Container
```

---

## 7. DaemonSet 모니터링

### 7.1 DaemonSet 상태 확인

**기본 상태 확인:**

```bash
# DaemonSet 목록
kubectl get daemonset -A

# 특정 DaemonSet 상세 정보
kubectl describe daemonset <name> -n <namespace>

# DaemonSet 상태 확인
kubectl get daemonset <name> -n <namespace> -o yaml
```

**상태 필드:**
- `desiredNumberScheduled`: 원하는 Pod 수 (노드 수)
- `currentNumberScheduled`: 현재 스케줄된 Pod 수
- `numberReady`: Ready 상태 Pod 수
- `numberAvailable`: 사용 가능한 Pod 수
- `numberUnavailable`: 사용 불가능한 Pod 수

### 7.2 Pod 상태 확인

**노드별 Pod 확인:**

```bash
# 모든 노드의 DaemonSet Pod 확인
kubectl get pods -n kube-system -l name=fluentd-logging -o wide

# 특정 노드의 Pod 확인
kubectl get pods -n kube-system -l name=fluentd-logging --field-selector spec.nodeName=<node-name>

# Pod 이벤트 확인
kubectl describe pod <pod-name> -n <namespace>
```

### 7.3 문제 해결

**Pod가 생성되지 않는 경우:**

```bash
# 1. DaemonSet 상태 확인
kubectl describe daemonset <name> -n <namespace>

# 2. 노드 라벨 확인
kubectl get nodes --show-labels

# 3. nodeSelector/nodeAffinity 확인
kubectl get daemonset <name> -n <namespace> -o yaml | grep -A 10 nodeSelector

# 4. tolerations 확인
kubectl get daemonset <name> -n <namespace> -o yaml | grep -A 10 tolerations
```

**Pod가 Ready 상태가 아닌 경우:**

```bash
# Pod 로그 확인
kubectl logs <pod-name> -n <namespace>

# Pod 이벤트 확인
kubectl describe pod <pod-name> -n <namespace>

# 리소스 부족 확인
kubectl top node
kubectl top pod -n <namespace>
```

---

## 8. Best Practices

### 8.1 리소스 제한 설정

**항상 requests/limits 설정:**

```yaml
spec:
  template:
    spec:
      containers:
      - name: fluentd
        resources:
          requests:
            cpu: "100m"
            memory: "200Mi"
          limits:
            cpu: "500m"
            memory: "500Mi"
```

**이유:**
- 노드 리소스 보호
- 다른 Pod와의 리소스 경합 방지
- OOMKilled 방지

### 8.2 노드 선택 전략

**필요한 노드에만 배포:**

```yaml
# Worker Node에만 배포
spec:
  template:
    spec:
      nodeSelector:
        node-role.kubernetes.io/worker: ""
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
```

**Master Node 포함 배포:**

```yaml
spec:
  template:
    spec:
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
```

### 8.3 업데이트 전략 선택

**안정적인 업데이트:**

```yaml
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1  # 한 번에 1개씩만 업데이트
```

**빠른 업데이트 (위험):**

```yaml
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 50%  # 절반씩 업데이트 (주의!)
```

### 8.4 네트워크 및 외부 통신

**기본 네트워크 동작:**

DaemonSet Pod는 기본적으로 **일반 Pod와 동일한 Pod 네트워크**를 사용합니다. 따라서:

```yaml
# 기본 설정 (hostNetwork 없음)
spec:
  template:
    spec:
      containers:
      - name: fluentd
        image: fluent/fluentd:latest
        # Pod 네트워크 사용 (외부 통신 제한 가능)
```

**특징:**
- Pod 네트워크를 사용하므로 **외부 통신이 제한될 수 있음**
- 클러스터 내부 통신은 가능 (Service, ClusterIP 등)
- 외부 API 호출이 필요한 경우 네트워크 정책 확인 필요

**외부 통신이 필요한 경우:**

**1. hostNetwork 사용 (노드 네트워크 직접 사용):**

```yaml
spec:
  template:
    spec:
      hostNetwork: true      # 노드 네트워크 직접 사용
      containers:
      - name: monitor
        image: prom/node-exporter:latest
```

**장점:**
- 노드의 네트워크 인터페이스 직접 접근
- 외부 통신 제한 없음
- 노드의 IP 주소 사용

**단점:**
- 보안 위험 증가
- Pod 포트 충돌 가능 (노드 포트 직접 사용)
- Service와의 통합 제한

**2. Service를 통한 외부 통신:**

```yaml
# DaemonSet Pod는 Service를 통해 외부 통신
apiVersion: v1
kind: Service
metadata:
  name: fluentd-service
spec:
  type: LoadBalancer  # 또는 NodePort
  selector:
    name: fluentd
  ports:
  - port: 80
    targetPort: 24224
```

**3. NetworkPolicy 확인:**

```yaml
# NetworkPolicy가 외부 통신을 막는 경우
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-external
spec:
  podSelector:
    matchLabels:
      name: fluentd
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector: {}  # 클러스터 내부만 허용
```

**권장사항:**
- **외부 통신이 필수인 경우**: `hostNetwork: true` 사용 (보안 고려)
- **클러스터 내부 통신만 필요한 경우**: 기본 Pod 네트워크 사용
- **외부 API 호출이 필요한 경우**: NetworkPolicy 확인 및 필요시 수정

### 8.5 호스트 리소스 접근

**필요한 경우에만 호스트 리소스 접근:**

```yaml
spec:
  template:
    spec:
      hostNetwork: true      # 호스트 네트워크 (필요시만)
      hostPID: true          # 호스트 PID 네임스페이스 (필요시만)
      containers:
      - name: monitor
        securityContext:
          privileged: true   # privileged 모드 (필요시만)
```

**보안 고려사항:**
- `hostNetwork`, `hostPID`, `privileged`는 보안 위험 증가
- 필요한 경우에만 사용
- RBAC으로 접근 제어

### 8.6 볼륨 마운트

**호스트 경로 마운트:**

```yaml
spec:
  template:
    spec:
      containers:
      - name: fluentd
        volumeMounts:
        - name: varlog
          mountPath: /var/log
          readOnly: true  # 읽기 전용으로 마운트
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
          type: Directory
```

**주의사항:**
- 필요한 경로만 마운트
- 가능하면 `readOnly: true` 사용
- 민감한 정보 접근 주의

---

## 9. DaemonSet 제한사항

### 9.1 PDB 적용 불가

**Pod Disruption Budget과 호환되지 않음:**

```yaml
# ❌ DaemonSet에는 PDB 적용 불가
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: fluentd-pdb
spec:
  minAvailable: 1
  selector:
    matchLabels:
      name: fluentd-logging
```

**이유:**
- DaemonSet은 노드별 1개 Pod만 실행
- PDB는 여러 Pod 중 최소 가용 수를 보장하는 용도
- DaemonSet은 이미 노드별 1개가 보장됨

### 9.2 HPA 적용 불가

**Horizontal Pod Autoscaler와 호환되지 않음:**

```yaml
# ❌ DaemonSet에는 HPA 적용 불가
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: fluentd-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: DaemonSet
    name: fluentd
  minReplicas: 1
  maxReplicas: 10
```

**이유:**
- DaemonSet Pod 수는 노드 수에 따라 자동 결정
- HPA는 메트릭 기반으로 Pod 수를 조정하는 용도
- DaemonSet은 노드 추가/제거로만 Pod 수가 변경됨

### 9.3 수동 스케일링 불가

**kubectl scale 명령어 사용 불가:**

```bash
# ❌ DaemonSet은 scale 불가
kubectl scale daemonset fluentd --replicas=5
```

**이유:**
- DaemonSet Pod 수는 노드 수에 따라 자동 결정
- 수동으로 스케일링할 수 없음

---

## 10. 실전 예시: 로깅 에이전트 DaemonSet

### 10.1 Fluentd DaemonSet 전체 예시

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
spec:
  selector:
    matchLabels:
      k8s-app: fluentd-logging
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  template:
    metadata:
      labels:
        k8s-app: fluentd-logging
    spec:
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      containers:
      - name: fluentd
        image: fluent/fluentd-kubernetes-daemonset:v1-debian-elasticsearch
        env:
        - name: FLUENT_ELASTICSEARCH_HOST
          value: "elasticsearch.logging.svc.cluster.local"
        - name: FLUENT_ELASTICSEARCH_PORT
          value: "9200"
        - name: FLUENT_ELASTICSEARCH_SCHEME
          value: "http"
        resources:
          requests:
            cpu: "100m"
            memory: "200Mi"
          limits:
            cpu: "500m"
            memory: "500Mi"
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
        - name: fluentd-config
          mountPath: /fluentd/etc
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
      - name: fluentd-config
        configMap:
          name: fluentd-config
```

### 10.2 ConfigMap 설정

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentd-config
  namespace: kube-system
data:
  fluent.conf: |
    <source>
      @type tail
      path /var/log/containers/*.log
      pos_file /var/log/fluentd-containers.log.pos
      tag kubernetes.*
      read_from_head true
      <parse>
        @type json
        time_key time
        time_format %Y-%m-%dT%H:%M:%S.%NZ
      </parse>
    </source>
    <match kubernetes.**>
      @type elasticsearch
      host "#{ENV['FLUENT_ELASTICSEARCH_HOST']}"
      port "#{ENV['FLUENT_ELASTICSEARCH_PORT']}"
      scheme "#{ENV['FLUENT_ELASTICSEARCH_SCHEME']}"
      logstash_format true
      logstash_prefix kubernetes
    </match>
```

---

## 마무리

**핵심 포인트:**

- **DaemonSet은 클러스터의 모든 노드(또는 특정 노드)에 정확히 1개의 Pod를 배포하는 리소스입니다.**
- **로깅, 모니터링, 네트워크 플러그인 등 모든 노드에서 실행되어야 하는 시스템 레벨 작업에 필수적입니다.**
- **Deployment/ReplicaSet과 달리 노드 수에 따라 Pod 수가 자동으로 결정되며, 수동 스케일링이 불가능합니다.**
- **nodeSelector, nodeAffinity, tolerations를 통해 특정 노드에만 배포할 수 있습니다.**
- **PDB, HPA와 호환되지 않으며, RollingUpdate 전략으로 무중단 업데이트가 가능합니다.**

DaemonSet을 적절히 활용하면 클러스터 전체에 걸쳐 일관된 시스템 레벨 작업을 자동화할 수 있습니다. 특히 로깅, 모니터링, 네트워크 플러그인 등은 DaemonSet 없이는 구현하기 어려운 작업입니다. 🚀

