---
layout: post
title: "Kubernetes IaC 도구 완전 정리: Kustomize, Helm, Terraform, Pulumi 특징과 사용 상황"
date: 2026-01-21
categories: [kubernetes, devops, infrastructure]
tags: [Kubernetes, IaC, InfrastructureAsCode, Kustomize, Helm, Terraform, Pulumi, DevOps, GitOps, 인프라자동화]
---

이전 글에서 Spring Boot와 NestJS의 DDD 프로젝트 구조 차이점을 비교했습니다. 이번 글에서는 **Kubernetes Infrastructure as Code (IaC) 도구들**의 특징과 각각을 언제 사용해야 하는지 정리해보겠습니다.

Kubernetes에서 인프라를 코드로 관리하는 방법은 여러 가지가 있습니다. 각 도구는 서로 다른 철학과 접근 방식을 가지고 있으며, 프로젝트의 요구사항에 따라 적절한 도구를 선택하는 것이 중요합니다.

---

## 1. Infrastructure as Code (IaC)란?

### 1.1 개념

**Infrastructure as Code (IaC)**는 인프라스트럭처를 코드로 정의하고 관리하는 방식입니다:

- **선언적 정의**: 원하는 상태를 선언하면 도구가 자동으로 적용
- **버전 관리**: Git으로 인프라 변경 이력 추적
- **재현 가능성**: 동일한 코드로 동일한 인프라 재생성
- **자동화**: 수동 작업 최소화

### 1.2 Kubernetes에서의 IaC

Kubernetes에서는 YAML 매니페스트 파일로 리소스를 정의합니다:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: my-app
        image: myapp:1.0.0
```

하지만 실제 프로젝트에서는:
- 여러 환경 (dev, staging, prod)
- 반복되는 패턴
- 복잡한 의존성 관리

이런 요구사항을 해결하기 위해 다양한 IaC 도구가 등장했습니다.

---

## 2. Kustomize

### 2.1 특징

**Kustomize**는 Kubernetes 네이티브 템플릿화 도구입니다:

- ✅ **Kubernetes 네이티브**: kubectl에 내장 (v1.14+)
- ✅ **YAML 기반**: 추가 언어 학습 불필요
- ✅ **선언적**: 원하는 상태만 선언
- ✅ **환경별 오버레이**: base + overlay 패턴

### 2.2 구조 예시

**실제 프로젝트 구조:**
```
k8s/
├── base/
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   └── service.yaml
├── overlays/
│   ├── dev/
│   │   ├── kustomization.yaml
│   │   └── patch.yaml
│   └── prod/
│       ├── kustomization.yaml
│       └── patch.yaml
```

**base/kustomization.yaml:**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- deployment.yaml
- service.yaml

commonLabels:
  app: my-app
  managed-by: kustomize
```

**overlays/dev/kustomization.yaml:**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
- ../../base

replicas:
- name: my-app
  count: 1  # 개발 환경은 1개만

images:
- name: my-app
  newTag: dev-latest

patches:
- path: patch.yaml
```

**overlays/prod/kustomization.yaml:**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
- ../../base

replicas:
- name: my-app
  count: 5  # 프로덕션은 5개

images:
- name: my-app
  newTag: v1.2.3

resources:
- ingress.yaml  # 프로덕션만 Ingress 추가
```

### 2.3 주요 기능

**1. 리소스 통합:**
```yaml
resources:
- deployment.yaml
- service.yaml
- configmap.yaml
```

**2. 이미지 태그 변경:**
```yaml
images:
- name: my-app
  newName: registry.example.com/my-app
  newTag: v1.2.3
```

**3. 리소스 패치:**
```yaml
patches:
- target:
    kind: Deployment
    name: my-app
  patch: |-
    - op: replace
      path: /spec/replicas
      value: 5
```

**4. 공통 라벨/어노테이션:**
```yaml
commonLabels:
  environment: production
  team: backend

commonAnnotations:
  description: "Production deployment"
```

### 2.4 사용 방법

```bash
# 빌드 (미리보기)
kubectl kustomize overlays/prod

# 빌드 및 적용
kubectl apply -k overlays/prod

# 또는 kustomize 직접 사용
kustomize build overlays/prod | kubectl apply -f -
```

### 2.5 장점

- ✅ **학습 곡선 낮음**: YAML만 알면 됨
- ✅ **Kubernetes 네이티브**: 추가 설치 불필요
- ✅ **환경 분리**: base + overlay로 깔끔한 구조
- ✅ **GitOps 친화적**: ArgoCD, Flux와 잘 통합

### 2.6 단점

- ❌ **재사용성 제한**: 다른 프로젝트로 차트 공유 어려움
- ❌ **복잡한 로직 부족**: 조건문, 반복문 등 제한적
- ❌ **의존성 관리 없음**: Helm처럼 차트 의존성 관리 불가

### 2.7 사용 상황

**Kustomize를 선택해야 하는 경우:**

- ✅ **간단한 프로젝트**: 복잡한 템플릿 로직이 필요 없을 때
- ✅ **Kubernetes 네이티브 선호**: 추가 도구 설치를 원하지 않을 때
- ✅ **환경별 설정 관리**: dev, staging, prod 환경 분리가 중요할 때
- ✅ **GitOps 파이프라인**: ArgoCD, Flux와 함께 사용할 때
- ✅ **팀이 YAML에 익숙**: 새로운 언어 학습을 원하지 않을 때

**실제 사용 예시:**
- 마이크로서비스 아키텍처에서 각 서비스별 매니페스트 관리
- 환경별 설정만 다른 동일한 애플리케이션 배포
- CI/CD 파이프라인에서 간단한 배포 자동화

---

## 3. Helm

### 3.1 특징

**Helm**은 Kubernetes용 패키지 매니저입니다:

- ✅ **차트 패키징**: 재사용 가능한 차트로 패키징
- ✅ **템플릿 엔진**: Go 템플릿으로 동적 생성
- ✅ **의존성 관리**: 다른 차트를 의존성으로 추가
- ✅ **버전 관리**: 차트 버전 및 릴리스 관리

### 3.2 구조 예시

```
my-chart/
├── Chart.yaml              # 차트 메타데이터
├── values.yaml             # 기본 값
├── values-dev.yaml         # 개발 환경 값
├── values-prod.yaml        # 프로덕션 환경 값
├── charts/                 # 의존성 차트
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    ├── ingress.yaml
    ├── configmap.yaml
    └── _helpers.tpl         # 헬퍼 템플릿
```

**Chart.yaml:**
```yaml
apiVersion: v2
name: my-app
description: My application Helm chart
type: application
version: 1.0.0
appVersion: "1.2.3"
dependencies:
  - name: postgresql
    version: "12.1.2"
    repository: "https://charts.bitnami.com/bitnami"
    condition: postgresql.enabled
```

**values.yaml:**
```yaml
replicaCount: 3

image:
  repository: myapp
  tag: "1.0.0"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: false
  className: "nginx"
  hosts:
    - host: myapp.example.com
      paths:
        - path: /
          pathType: Prefix

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80
```

**templates/deployment.yaml:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-app.fullname" . }}
  labels:
    {{- include "my-app.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "my-app.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "my-app.labels" . | nindent 8 }}
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        ports:
        - containerPort: 8080
        resources:
          {{- toYaml .Values.resources | nindent 10 }}
        {{- if .Values.env }}
        env:
        {{- range .Values.env }}
        - name: {{ .name }}
          value: {{ .value | quote }}
        {{- end }}
        {{- end }}
```

**templates/_helpers.tpl:**
```yaml
{{- define "my-app.fullname" -}}
{{- printf "%s-%s" .Release.Name .Chart.Name | trunc 63 | trimSuffix "-" }}
{{- end }}

{{- define "my-app.labels" -}}
helm.sh/chart: {{ .Chart.Name }}-{{ .Chart.Version }}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

### 3.3 주요 기능

**1. 템플릿 변수:**
```yaml
image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

**2. 조건문:**
```yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
# ...
{{- end }}
```

**3. 반복문:**
```yaml
{{- range .Values.env }}
- name: {{ .name }}
  value: {{ .value }}
{{- end }}
```

**4. 함수:**
```yaml
{{ .Values.replicaCount | default 3 }}
{{ include "my-app.fullname" . }}
```

### 3.4 사용 방법

```bash
# 차트 설치
helm install my-release ./my-chart -f values-prod.yaml

# 업그레이드
helm upgrade my-release ./my-chart -f values-prod.yaml

# 롤백
helm rollback my-release 1

# 릴리스 목록
helm list

# 차트 패키징
helm package ./my-chart

# 차트 저장소에 푸시
helm push my-chart-1.0.0.tgz oci://registry.example.com/charts
```

### 3.5 장점

- ✅ **재사용성**: 차트를 다른 프로젝트에서 재사용 가능
- ✅ **의존성 관리**: 다른 차트를 의존성으로 추가
- ✅ **버전 관리**: 차트 버전 및 릴리스 히스토리
- ✅ **풍부한 템플릿 기능**: 조건문, 반복문, 함수 등
- ✅ **대규모 생태계**: 수천 개의 공개 차트

### 3.6 단점

- ❌ **학습 곡선**: Go 템플릿 문법 학습 필요
- ❌ **복잡성**: 간단한 프로젝트에는 과함
- ❌ **디버깅 어려움**: 템플릿 오류 추적이 어려울 수 있음

### 3.7 사용 상황

**Helm을 선택해야 하는 경우:**

- ✅ **재사용 가능한 패키지 필요**: 여러 프로젝트에서 동일한 설정 재사용
- ✅ **복잡한 템플릿 로직**: 조건문, 반복문 등이 필요할 때
- ✅ **의존성 관리**: PostgreSQL, Redis 등 외부 차트 의존성 필요
- ✅ **차트 배포**: 팀 내부 또는 공개 차트 저장소에 배포
- ✅ **버전 관리**: 차트 버전 및 릴리스 히스토리 관리 중요

**실제 사용 예시:**
- 공통 애플리케이션 패턴을 차트로 패키징하여 여러 팀에서 재사용
- 복잡한 설정 옵션을 가진 애플리케이션 배포
- Helm Hub나 Artifact Hub에서 공개 차트 사용

---

## 4. Terraform

### 4.1 특징

**Terraform**은 멀티 클라우드 Infrastructure as Code 도구입니다:

- ✅ **멀티 클라우드**: AWS, GCP, Azure, Kubernetes 등 지원
- ✅ **상태 관리**: 실제 인프라 상태를 State 파일로 관리
- ✅ **계획 단계**: `terraform plan`으로 변경 사항 미리 확인
- ✅ **모듈화**: 재사용 가능한 모듈로 구조화

### 4.2 구조 예시

```
terraform/
├── main.tf
├── variables.tf
├── outputs.tf
├── terraform.tfstate      # 상태 파일 (Git에 포함하지 않음)
├── terraform.tfvars       # 변수 값
└── modules/
    └── kubernetes/
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```

**main.tf:**
```hcl
terraform {
  required_version = ">= 1.0"
  
  required_providers {
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.20"
    }
  }
  
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "kubernetes/terraform.tfstate"
    region = "us-west-2"
  }
}

provider "kubernetes" {
  config_path = "~/.kube/config"
}

# Namespace
resource "kubernetes_namespace" "app" {
  metadata {
    name = var.namespace
  }
}

# Deployment
resource "kubernetes_deployment" "app" {
  metadata {
    name      = var.app_name
    namespace = kubernetes_namespace.app.metadata[0].name
    labels = {
      app = var.app_name
    }
  }
  
  spec {
    replicas = var.replicas
    
    selector {
      match_labels = {
        app = var.app_name
      }
    }
    
    template {
      metadata {
        labels = {
          app = var.app_name
        }
      }
      
      spec {
        container {
          name  = var.app_name
          image = "${var.image_repository}:${var.image_tag}"
          
          port {
            container_port = var.container_port
          }
          
          resources {
            requests = {
              cpu    = var.cpu_request
              memory = var.memory_request
            }
            limits = {
              cpu    = var.cpu_limit
              memory = var.memory_limit
            }
          }
          
          dynamic "env" {
            for_each = var.env_vars
            content {
              name  = env.value.name
              value = env.value.value
            }
          }
        }
      }
    }
  }
}

# Service
resource "kubernetes_service" "app" {
  metadata {
    name      = var.app_name
    namespace = kubernetes_namespace.app.metadata[0].name
  }
  
  spec {
    selector = {
      app = var.app_name
    }
    
    port {
      port        = 80
      target_port = var.container_port
    }
    
    type = var.service_type
  }
  
  depends_on = [kubernetes_deployment.app]
}
```

**variables.tf:**
```hcl
variable "namespace" {
  type        = string
  description = "Kubernetes namespace"
  default     = "default"
}

variable "app_name" {
  type        = string
  description = "Application name"
}

variable "replicas" {
  type        = number
  description = "Number of replicas"
  default     = 3
}

variable "image_repository" {
  type        = string
  description = "Docker image repository"
}

variable "image_tag" {
  type        = string
  description = "Docker image tag"
  default     = "latest"
}

variable "container_port" {
  type        = number
  description = "Container port"
  default     = 8080
}

variable "cpu_request" {
  type        = string
  description = "CPU request"
  default     = "100m"
}

variable "memory_request" {
  type        = string
  description = "Memory request"
  default     = "128Mi"
}

variable "cpu_limit" {
  type        = string
  description = "CPU limit"
  default     = "500m"
}

variable "memory_limit" {
  type        = string
  description = "Memory limit"
  default     = "512Mi"
}

variable "service_type" {
  type        = string
  description = "Service type"
  default     = "ClusterIP"
}

variable "env_vars" {
  type = list(object({
    name  = string
    value = string
  }))
  description = "Environment variables"
  default     = []
}
```

**outputs.tf:**
```hcl
output "deployment_name" {
  value       = kubernetes_deployment.app.metadata[0].name
  description = "Deployment name"
}

output "service_name" {
  value       = kubernetes_service.app.metadata[0].name
  description = "Service name"
}

output "namespace" {
  value       = kubernetes_namespace.app.metadata[0].name
  description = "Namespace"
}
```

**terraform.tfvars:**
```hcl
namespace         = "production"
app_name          = "my-app"
replicas          = 5
image_repository   = "myapp"
image_tag          = "v1.2.3"
container_port     = 8080
cpu_request        = "200m"
memory_request     = "256Mi"
cpu_limit          = "1000m"
memory_limit       = "512Mi"
service_type       = "ClusterIP"

env_vars = [
  {
    name  = "ENV"
    value = "production"
  },
  {
    name  = "LOG_LEVEL"
    value = "info"
  }
]
```

### 4.3 주요 기능

**1. 상태 관리:**
```bash
# 로컬 상태 파일
terraform.tfstate

# 원격 상태 (S3, GCS 등)
terraform {
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "kubernetes/terraform.tfstate"
  }
}
```

**2. 계획 단계:**
```bash
terraform plan
# 변경 사항을 미리 확인
```

**3. 모듈화:**
```hcl
module "kubernetes_app" {
  source = "./modules/kubernetes"
  
  app_name         = "my-app"
  replicas         = 3
  image_repository = "myapp"
  image_tag        = "v1.2.3"
}
```

**4. 동적 블록:**
```hcl
dynamic "env" {
  for_each = var.env_vars
  content {
    name  = env.value.name
    value = env.value.value
  }
}
```

### 4.4 사용 방법

```bash
# 초기화
terraform init

# 계획 확인
terraform plan

# 적용
terraform apply

# 특정 리소스만 적용
terraform apply -target=kubernetes_deployment.app

# 상태 확인
terraform show

# 삭제
terraform destroy

# 상태 가져오기
terraform import kubernetes_deployment.app default/my-app
```

### 4.5 장점

- ✅ **멀티 클라우드**: Kubernetes뿐만 아니라 클라우드 리소스도 관리
- ✅ **상태 관리**: 실제 인프라 상태 추적
- ✅ **계획 단계**: 변경 사항 미리 확인
- ✅ **모듈화**: 재사용 가능한 모듈
- ✅ **생태계**: 다양한 Provider 지원

### 4.6 단점

- ❌ **학습 곡선**: HCL 문법 학습 필요
- ❌ **상태 파일 관리**: State 파일 동기화 중요
- ❌ **Kubernetes 전용 아님**: Kubernetes만 관리할 때는 과함

### 4.7 사용 상황

**Terraform을 선택해야 하는 경우:**

- ✅ **멀티 클라우드**: Kubernetes + AWS/GCP/Azure 리소스 함께 관리
- ✅ **클라우드 리소스**: VPC, Load Balancer, Database 등도 관리
- ✅ **상태 관리 중요**: 실제 인프라 상태 추적이 필요할 때
- ✅ **계획 단계 필요**: 변경 사항을 미리 확인하고 싶을 때
- ✅ **기존 Terraform 사용**: 이미 Terraform을 사용 중일 때

**실제 사용 예시:**
- Kubernetes 클러스터 + AWS RDS, S3 등 클라우드 리소스 함께 관리
- 멀티 클라우드 환경에서 일관된 IaC 도구 사용
- 인프라 변경 이력 및 상태 추적이 중요한 엔터프라이즈 환경

---

## 5. Pulumi

### 5.1 특징

**Pulumi**는 일반 프로그래밍 언어로 인프라를 정의합니다:

- ✅ **일반 프로그래밍 언어**: Python, TypeScript, Go, C# 등
- ✅ **타입 안정성**: 컴파일 타임 타입 체크
- ✅ **IDE 지원**: 자동완성, 리팩토링 등
- ✅ **테스트 가능**: 일반 테스트 프레임워크 사용

### 5.2 구조 예시 (TypeScript)

```
pulumi/
├── Pulumi.yaml
├── Pulumi.dev.yaml
├── Pulumi.prod.yaml
├── package.json
└── index.ts
```

**Pulumi.yaml:**
```yaml
name: my-app
runtime: nodejs
description: My application infrastructure
```

**Pulumi.dev.yaml:**
```yaml
config:
  kubernetes:replicas: "1"
  kubernetes:imageTag: "dev-latest"
```

**index.ts:**
```typescript
import * as k8s from "@pulumi/kubernetes";
import * as pulumi from "@pulumi/pulumi";

const config = new pulumi.Config();

const appName = "my-app";
const replicas = config.getNumber("replicas") || 3;
const imageTag = config.get("imageTag") || "latest";

// Namespace
const namespace = new k8s.core.v1.Namespace(appName, {
  metadata: {
    name: appName,
  },
});

// Deployment
const deployment = new k8s.apps.v1.Deployment(appName, {
  metadata: {
    name: appName,
    namespace: namespace.metadata.name,
    labels: {
      app: appName,
    },
  },
  spec: {
    replicas: replicas,
    selector: {
      matchLabels: {
        app: appName,
      },
    },
    template: {
      metadata: {
        labels: {
          app: appName,
        },
      },
      spec: {
        containers: [
          {
            name: appName,
            image: `myapp:${imageTag}`,
            ports: [
              {
                containerPort: 8080,
              },
            ],
            resources: {
              requests: {
                cpu: "100m",
                memory: "128Mi",
              },
              limits: {
                cpu: "500m",
                memory: "512Mi",
              },
            },
            env: [
              {
                name: "ENV",
                value: pulumi.getStack(),
              },
            ],
          },
        ],
      },
    },
  },
});

// Service
const service = new k8s.core.v1.Service(appName, {
  metadata: {
    name: appName,
    namespace: namespace.metadata.name,
  },
  spec: {
    selector: {
      app: appName,
    },
    ports: [
      {
        port: 80,
        targetPort: 8080,
      },
    ],
    type: "ClusterIP",
  },
});

// Outputs
export const deploymentName = deployment.metadata.name;
export const serviceName = service.metadata.name;
export const namespaceName = namespace.metadata.name;
```

### 5.3 Python 예시

```python
import pulumi
from pulumi_kubernetes.apps.v1 import Deployment
from pulumi_kubernetes.core.v1 import Service, Namespace

config = pulumi.Config()

app_name = "my-app"
replicas = config.get_int("replicas") or 3
image_tag = config.get("imageTag") or "latest"

# Namespace
namespace = Namespace(
    app_name,
    metadata={
        "name": app_name,
    }
)

# Deployment
deployment = Deployment(
    app_name,
    metadata={
        "name": app_name,
        "namespace": namespace.metadata["name"],
        "labels": {
            "app": app_name,
        },
    },
    spec={
        "replicas": replicas,
        "selector": {
            "match_labels": {
                "app": app_name,
            },
        },
        "template": {
            "metadata": {
                "labels": {
                    "app": app_name,
                },
            },
            "spec": {
                "containers": [{
                    "name": app_name,
                    "image": f"myapp:{image_tag}",
                    "ports": [{
                        "container_port": 8080,
                    }],
                    "resources": {
                        "requests": {
                            "cpu": "100m",
                            "memory": "128Mi",
                        },
                        "limits": {
                            "cpu": "500m",
                            "memory": "512Mi",
                        },
                    },
                }],
            },
        },
    },
)

# Service
service = Service(
    app_name,
    metadata={
        "name": app_name,
        "namespace": namespace.metadata["name"],
    },
    spec={
        "selector": {
            "app": app_name,
        },
        "ports": [{
            "port": 80,
            "target_port": 8080,
        }],
        "type": "ClusterIP",
    },
)

# Outputs
pulumi.export("deployment_name", deployment.metadata["name"])
pulumi.export("service_name", service.metadata["name"])
```

### 5.4 주요 기능

**1. 타입 안정성:**
```typescript
const deployment = new k8s.apps.v1.Deployment("app", {
  // IDE가 자동완성 및 타입 체크 제공
  spec: {
    replicas: 3,  // number 타입
    // ...
  },
});
```

**2. 조건문 및 반복문:**
```typescript
const containers = config.getBoolean("enableSidecar")
  ? [mainContainer, sidecarContainer]
  : [mainContainer];

deployment.spec.template.spec.containers = containers;
```

**3. 함수 및 모듈화:**
```typescript
function createDeployment(name: string, image: string) {
  return new k8s.apps.v1.Deployment(name, {
    // ...
  });
}
```

**4. 테스트:**
```typescript
import * as assert from "assert";

// 단위 테스트
describe("Deployment", () => {
  it("should have correct replicas", () => {
    const deployment = createDeployment("app", "myapp:1.0.0");
    assert.equal(deployment.spec.replicas, 3);
  });
});
```

### 5.5 사용 방법

```bash
# 초기화
pulumi new kubernetes-typescript

# 스택 생성
pulumi stack init dev

# 설정
pulumi config set replicas 3
pulumi config set imageTag v1.2.3

# 미리보기
pulumi preview

# 배포
pulumi up

# 삭제
pulumi destroy

# 스택 목록
pulumi stack ls
```

### 5.6 장점

- ✅ **일반 프로그래밍 언어**: Python, TypeScript 등 친숙한 언어
- ✅ **타입 안정성**: 컴파일 타임 오류 발견
- ✅ **IDE 지원**: 자동완성, 리팩토링 등
- ✅ **테스트 가능**: 일반 테스트 프레임워크 사용
- ✅ **재사용성**: 함수, 클래스 등으로 구조화

### 5.7 단점

- ❌ **학습 곡선**: Pulumi SDK 학습 필요
- ❌ **상태 관리**: Pulumi Cloud 또는 자체 백엔드 필요
- ❌ **커뮤니티**: Helm, Terraform보다 작은 생태계

### 5.8 사용 상황

**Pulumi를 선택해야 하는 경우:**

- ✅ **프로그래밍 언어 선호**: YAML/HCL 대신 Python/TypeScript 선호
- ✅ **타입 안정성 중요**: 컴파일 타임 오류 발견
- ✅ **복잡한 로직**: 조건문, 반복문, 함수 등이 많이 필요
- ✅ **테스트 중요**: 인프라 코드를 테스트하고 싶을 때
- ✅ **IDE 지원**: 자동완성, 리팩토링 등이 중요할 때

**실제 사용 예시:**
- 개발팀이 Python/TypeScript에 익숙한 경우
- 복잡한 비즈니스 로직이 인프라 정의에 포함되는 경우
- 인프라 코드를 테스트하고 싶은 경우

---

## 6. 도구 비교표

| 항목 | Kustomize | Helm | Terraform | Pulumi |
|------|-----------|------|-----------|--------|
| **언어** | YAML | YAML + Go 템플릿 | HCL | Python/TS/Go/C# |
| **학습 곡선** | 낮음 | 중간 | 중간 | 높음 |
| **재사용성** | 중간 | 높음 | 높음 | 높음 |
| **환경 분리** | ✅ (overlay) | ✅ (values) | ✅ (workspace) | ✅ (stack) |
| **의존성 관리** | ❌ | ✅ | ✅ | ✅ |
| **상태 관리** | ❌ | ✅ (Helm) | ✅ (State) | ✅ (Pulumi Cloud) |
| **멀티 클라우드** | ❌ | ❌ | ✅ | ✅ |
| **템플릿 기능** | 제한적 | 강력 | 강력 | 매우 강력 |
| **타입 안정성** | ❌ | ❌ | 부분적 | ✅ |
| **IDE 지원** | 제한적 | 제한적 | 부분적 | ✅ |
| **테스트** | 어려움 | 어려움 | 어려움 | 가능 |
| **GitOps** | ✅ | ✅ | ✅ | ✅ |

---

## 7. 선택 가이드

### 7.1 프로젝트 규모별

**소규모 프로젝트 (1-5 서비스):**
- **추천**: Kustomize
- **이유**: 간단하고 Kubernetes 네이티브

**중규모 프로젝트 (5-20 서비스):**
- **추천**: Helm 또는 Kustomize
- **이유**: 재사용성과 환경 관리 중요

**대규모 프로젝트 (20+ 서비스):**
- **추천**: Helm 또는 Terraform
- **이유**: 모듈화 및 의존성 관리 중요

### 7.2 요구사항별

**Kubernetes만 관리:**
- Kustomize 또는 Helm

**Kubernetes + 클라우드 리소스:**
- Terraform 또는 Pulumi

**재사용 가능한 패키지:**
- Helm

**프로그래밍 언어 선호:**
- Pulumi

**간단함 우선:**
- Kustomize

### 7.3 팀 역량별

**YAML에 익숙:**
- Kustomize 또는 Helm

**프로그래밍에 익숙:**
- Pulumi

**DevOps 경험 풍부:**
- Terraform

---

## 8. 실제 프로젝트 예시

### 8.1 Kustomize 사용 예시 (현재 프로젝트)

```bash
# 환경 배포
kubectl apply -k k8s/environments

# 애플리케이션 배포
kubectl apply -k k8s/applications/backend

# 이미지 태그 변경
kustomize edit set image egovframe/egovframe-msa-edu-backend-apigateway:latest=v1.2.3
kubectl apply -k k8s/applications/backend
```

### 8.2 Helm 사용 예시

```bash
# 차트 설치
helm install my-app ./my-chart -f values-prod.yaml

# 업그레이드
helm upgrade my-app ./my-chart -f values-prod.yaml --set image.tag=v1.2.3

# 롤백
helm rollback my-app 1
```

### 8.3 Terraform 사용 예시

```bash
# 초기화
terraform init

# 계획
terraform plan -var-file=prod.tfvars

# 적용
terraform apply -var-file=prod.tfvars
```

### 8.4 Pulumi 사용 예시

```bash
# 스택 생성
pulumi stack init prod

# 설정
pulumi config set imageTag v1.2.3

# 배포
pulumi up
```

---

## 9. Best Practice

### 9.1 공통 Best Practice

1. **버전 관리**: 모든 코드를 Git으로 관리
2. **환경 분리**: dev, staging, prod 환경 명확히 분리
3. **DRY 원칙**: 반복되는 패턴은 재사용 가능하게
4. **문서화**: README와 주석으로 설명
5. **검증**: CI/CD에서 자동 검증

### 9.2 Kustomize Best Practice

```yaml
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- deployment.yaml
- service.yaml

commonLabels:
  app: my-app
  managed-by: kustomize

# overlays/prod/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
- ../../base

replicas:
- name: my-app
  count: 5

images:
- name: my-app
  newTag: v1.2.3
```

### 9.3 Helm Best Practice

```yaml
# Chart.yaml
apiVersion: v2
name: my-app
version: 1.0.0
description: My application

# values.yaml
replicaCount: 3
image:
  repository: myapp
  tag: "1.0.0"

# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-app.fullname" . }}
spec:
  replicas: {{ .Values.replicaCount }}
  # ...
```

### 9.4 Terraform Best Practice

```hcl
# variables.tf
variable "replicas" {
  type        = number
  description = "Number of replicas"
  default     = 3
}

# main.tf
resource "kubernetes_deployment" "app" {
  spec {
    replicas = var.replicas
    # ...
  }
}

# terraform.tfvars
replicas = 5
```

### 9.5 Pulumi Best Practice

```typescript
// index.ts
const config = new pulumi.Config();
const replicas = config.getNumber("replicas") || 3;

const deployment = new k8s.apps.v1.Deployment("app", {
  spec: {
    replicas: replicas,
    // ...
  },
});

// Pulumi.prod.yaml
config:
  replicas: "5"
```

---

## 마무리

**핵심 포인트:**

1. **Kustomize**: 간단하고 Kubernetes 네이티브, 환경 분리에 강점
2. **Helm**: 재사용 가능한 차트, 의존성 관리, 대규모 생태계
3. **Terraform**: 멀티 클라우드, 상태 관리, 계획 단계
4. **Pulumi**: 프로그래밍 언어, 타입 안정성, IDE 지원

**선택 기준:**

- **간단한 프로젝트** → Kustomize
- **재사용성 중요** → Helm
- **멀티 클라우드** → Terraform
- **프로그래밍 언어 선호** → Pulumi

프로젝트의 규모, 요구사항, 팀의 역량에 따라 적절한 도구를 선택하는 것이 중요합니다. 때로는 여러 도구를 조합하여 사용하는 것도 좋은 방법입니다 (예: Helm + Terraform). 🚀

다음 글에서는 **Kafka Producer의 동기/비동기 성능 비교**를 Spring Kafka 3.3.11 기준으로 정리해보겠습니다.
