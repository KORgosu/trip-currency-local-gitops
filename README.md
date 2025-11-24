# Trip Currency Service - GitOps Repository

여행 환율 서비스 플랫폼의 쿠버네티스 매니페스트 및 GitOps 설정을 관리하는 저장소입니다.

## 목차

- [개요](#개요)
- [아키텍처](#아키텍처)
- [디렉토리 구조](#디렉토리-구조)
- [서비스 구성](#서비스-구성)
- [인프라 구성요소](#인프라-구성요소)
- [환경 관리](#환경-관리)
- [배포 가이드](#배포-가이드)
- [모니터링 및 관찰성](#모니터링-및-관찰성)
- [리소스 관리](#리소스-관리)

---

## 개요

이 저장소는 **GitOps** 방식으로 쿠버네티스 클러스터에 배포되는 Trip Currency Service의 모든 매니페스트를 관리합니다.

### 주요 특징

- **Kustomize 기반 관리**: base와 overlay 구조로 환경별 설정 분리
- **다중 환경 지원**:
  - **Dev**: 로컬 개발 환경
  - **Staging**: 로컬 스테이징 환경
  - **Prod (Local)**: 로컬 PC 프로덕션 환경 (테스트 및 검증용)
  - **EKS (Cloud)**: AWS EKS 실제 프로덕션 환경
- **마이크로서비스 아키텍처**: 독립적으로 확장 가능한 서비스들
- **자동 확장**: HPA(Horizontal Pod Autoscaler)를 통한 트래픽 기반 자동 확장
- **클라우드 네이티브**: AWS EKS, ECR, ALB, EBS 등 AWS 리소스 통합
- **모니터링 준비**: Prometheus 메트릭 수집 어노테이션 포함

### GitOps 워크플로우

```
코드 변경 → Git Push → ArgoCD 감지 → 자동 배포 → 클러스터 동기화
```

---

## 아키텍처

### 시스템 구성도

```
┌─────────────────────────────────────────────────────────────┐
│                      Ingress Controller                      │
│                    (NGINX Ingress)                           │
└─────────────────────────┬───────────────────────────────────┘
                          │
          ┌───────────────┼───────────────┐
          │               │               │
          ▼               ▼               ▼
    ┌─────────┐    ┌──────────┐    ┌──────────┐
    │Frontend │    │ Currency │    │ History  │
    │ Service │    │ Service  │    │ Service  │
    └─────────┘    └────┬─────┘    └────┬─────┘
                        │               │
          ┌─────────────┼───────────────┘
          │             │
          ▼             ▼
    ┌─────────┐   ┌─────────┐
    │  MySQL  │   │ MongoDB │
    └─────────┘   └─────────┘
          │             │
          └──────┬──────┘
                 │
          ┌──────▼──────┐
          │    Kafka    │
          └─────────────┘
                 ▲
                 │
          ┌──────┴──────┐
          │ DataIngestor│
          │  (CronJob)  │
          └─────────────┘
```

### 서비스 간 통신

- **Frontend** → API 게이트웨이를 통해 백엔드 서비스 호출
- **Currency Service** → MySQL에 환율 데이터 저장
- **History Service** → MySQL에서 히스토리 데이터 조회/저장
- **Ranking Service** → MongoDB에 랭킹 데이터 저장/조회
- **DataIngestor** → 외부 API에서 데이터 수집 후 Kafka로 발행

### 배포 환경 비교

#### 로컬 환경 (Dev/Staging/Prod Overlays)
```
┌─────────────────────────────────┐
│      로컬 쿠버네티스 클러스터        │
│   (Docker Desktop/Minikube)     │
├─────────────────────────────────┤
│  NGINX Ingress Controller       │
│  MetalLB (LoadBalancer)         │
│  local-path Storage             │
│  Docker Hub 이미지               │
└─────────────────────────────────┘
```

#### AWS EKS 환경 (EKS Overlay)
```
┌─────────────────────────────────┐
│        AWS EKS 클러스터          │
│     (ap-northeast-2)            │
├─────────────────────────────────┤
│  AWS ALB Ingress Controller     │
│  AWS NLB (Frontend)             │
│  AWS EBS Storage (gp3-encrypted)│
│  AWS ECR 이미지                  │
│  AWS ACM SSL/TLS 인증서          │
│  도메인: 2025teamproject.store   │
└─────────────────────────────────┘
```

---

## 디렉토리 구조

```
trip-currency-local-gitops/
├── k8s/
│   ├── base/                           # 기본 매니페스트
│   │   ├── configmap.yaml             # 전역 ConfigMap
│   │   ├── secrets.yaml               # 전역 Secrets
│   │   ├── kustomization.yaml         # Base Kustomization
│   │   │
│   │   ├── ingress-controller/        # NGINX Ingress Controller
│   │   │   ├── nginx-controller.yaml
│   │   │   ├── rbac.yaml
│   │   │   └── kustomization.yaml
│   │   │
│   │   ├── metallb/                   # MetalLB 설정 (로컬 환경용)
│   │   │   ├── ipaddresspool.yaml
│   │   │   └── kustomization.yaml
│   │   │
│   │   ├── mysql/                     # MySQL Database
│   │   │   ├── statefulset.yaml       # StatefulSet (데이터 영구 저장)
│   │   │   ├── service.yaml           # Headless Service
│   │   │   ├── configmap.yaml
│   │   │   └── kustomization.yaml
│   │   │
│   │   ├── mongodb/                   # MongoDB Database
│   │   │   ├── statefulset.yaml       # StatefulSet (데이터 영구 저장)
│   │   │   ├── service.yaml           # Headless Service
│   │   │   ├── configmap.yaml
│   │   │   └── kustomization.yaml
│   │   │
│   │   ├── redis/                     # Redis Cache
│   │   │   ├── deployment.yaml        # Deployment (캐싱 전용)
│   │   │   ├── statefulset.yaml       # 예비용 (필요시 사용)
│   │   │   ├── service.yaml
│   │   │   └── kustomization.yaml
│   │   │
│   │   ├── kafka/                     # Kafka Message Queue
│   │   │   ├── zookeeper.yaml        # Kafka 의존성
│   │   │   ├── kafka.yaml
│   │   │   ├── kafka-ui.yaml         # Kafka 관리 UI
│   │   │   └── kustomization.yaml
│   │   │
│   │   └── services/                  # 애플리케이션 서비스
│   │       ├── currency-service/
│   │       │   ├── deployment.yaml
│   │       │   ├── service.yaml
│   │       │   ├── hpa.yaml
│   │       │   └── kustomization.yaml
│   │       │
│   │       ├── history-service/
│   │       │   ├── deployment.yaml
│   │       │   ├── service.yaml
│   │       │   ├── hpa.yaml
│   │       │   └── kustomization.yaml
│   │       │
│   │       ├── ranking-service/
│   │       │   ├── deployment.yaml
│   │       │   ├── service.yaml
│   │       │   ├── hpa.yaml
│   │       │   └── kustomization.yaml
│   │       │
│   │       ├── dataingestor-service/
│   │       │   ├── cronjob.yaml
│   │       │   └── kustomization.yaml
│   │       │
│   │       └── frontend/
│   │           ├── deployment.yaml
│   │           ├── service.yaml
│   │           ├── ingress.yaml
│   │           ├── configmap.yaml
│   │           ├── hpa.yaml
│   │           └── kustomization.yaml
│   │
│   └── overlays/                       # 환경별 오버레이
│       ├── dev/                       # 개발 환경
│       │   ├── kustomization.yaml
│       │   ├── namespace.yaml
│       │   ├── configmap.yaml
│       │   ├── ingress.yaml
│       │   └── ingress-patch.yaml
│       │
│       ├── staging/                   # 스테이징 환경
│       │   └── namespace.yaml
│       │
│       ├── prod/                      # 프로덕션 환경
│       │   ├── kustomization.yaml
│       │   ├── namespace.yaml
│       │   ├── ingress.yaml
│       │   └── ingress-patch.yaml
│       │
│       └── eks/                       # AWS EKS 환경
│           ├── kustomization.yaml
│           ├── ingress.yaml
│           └── storageclass.yaml
│
└── README.md
```

---

## 서비스 구성

### 1. Currency Service

**역할**: 환율 데이터 조회 및 관리

- **이미지**: `korgosu/service-currency:latest`
- **포트**: 8000
- **데이터베이스**: MySQL (`currency_db`)
- **헬스체크**: `/health`
- **리소스**:
  - Requests: 256Mi RAM, 100m CPU
  - Limits: 384Mi RAM, 500m CPU
- **오토스케일링**: CPU 70%, Memory 90% 기준 (1-10 pods)

### 2. History Service

**역할**: 환율 히스토리 데이터 관리

- **이미지**: `korgosu/service-history:latest`
- **포트**: 8000
- **데이터베이스**: MySQL (`currency_db`)
- **헬스체크**: `/health`
- **리소스**:
  - Requests: 256Mi RAM, 100m CPU
  - Limits: 384Mi RAM, 500m CPU
- **오토스케일링**: CPU 70%, Memory 90% 기준 (1-10 pods)

### 3. Ranking Service

**역할**: 환율 랭킹 및 통계 데이터 관리

- **이미지**: `korgosu/service-ranking:latest`
- **포트**: 8000
- **데이터베이스**: MongoDB
- **헬스체크**: `/health`
- **리소스**:
  - Requests: 256Mi RAM, 100m CPU
  - Limits: 384Mi RAM, 500m CPU
- **오토스케일링**: CPU 70%, Memory 90% 기준 (1-10 pods)
- **특이사항**: MongoDB 초기화 대기 시간이 길어 liveness probe 지연 시간 증가

### 4. DataIngestor Service (CronJob)

**역할**: 주기적인 외부 환율 데이터 수집

- **이미지**: `korgosu/service-dataingestor:latest`
- **실행 주기**: 5분마다 (`*/5 * * * *`)
- **동시 실행 정책**: `Forbid` (이전 Job 완료 시까지 대기)
- **타임아웃**: 10분 (600초)
- **재시도**: 최대 2회
- **리소스**:
  - Requests: 256Mi RAM, 100m CPU
  - Limits: 512Mi RAM, 500m CPU

### 5. Frontend Service

**역할**: React 기반 웹 UI

- **이미지**: `korgosu/service-frontend:latest`
- **포트**: 80
- **리소스**:
  - Requests: 32Mi RAM, 50m CPU
  - Limits: 64Mi RAM, 200m CPU
- **오토스케일링**: CPU 70%, Memory 80% 기준 (1-10 pods)
- **환경 설정**: `frontend-config` ConfigMap에서 API URL 로드

---

## 인프라 구성요소

### 데이터베이스

#### MySQL
- **용도**: 환율 및 히스토리 데이터 저장
- **구성**: StatefulSet (데이터 영구 저장)
- **이미지**: `mysql:8.0`
- **포트**: 3306
- **서비스**: Headless Service (`service-mysql`)
- **스토리지**: 10Gi PVC (ReadWriteOnce, `volumeClaimTemplates`로 자동 생성)
- **StorageClass**: `gp3-encrypted` (EKS), 기본값 (로컬)
- **데이터베이스**: `currency_db`
- **사용자**: `trip_user`
- **특징**: Pod 이름이 `mysql-0`로 고정되어 안정적인 네트워크 식별자 제공

#### MongoDB
- **용도**: 랭킹 및 통계 데이터 저장
- **구성**: StatefulSet (데이터 영구 저장)
- **이미지**: `mongo:6.0`
- **포트**: 27017
- **서비스**: Headless Service (`service-mongodb`)
- **스토리지**: 10Gi PVC (ReadWriteOnce, `volumeClaimTemplates`로 자동 생성)
- **StorageClass**: `gp3-encrypted` (EKS), 기본값 (로컬)
- **데이터베이스**: `currency_db`
- **특징**: Pod 이름이 `mongodb-0`로 고정되어 안정적인 네트워크 식별자 제공

#### Redis
- **용도**: 캐싱 및 세션 관리
- **구성**: Deployment (캐싱 전용, 데이터 손실 허용)
- **이미지**: `redis:7.0-alpine`
- **포트**: 6379
- **서비스**: ClusterIP Service (`service-redis`)
- **스토리지**: 임시 스토리지 (`emptyDir`, 200Mi 제한)
- **리소스**:
  - Requests: 128Mi RAM, 100m CPU
  - Limits: 256Mi RAM, 200m CPU
- **설정**: AOF 활성화, LRU 메모리 정책 (100mb 제한)
- **참고**: StatefulSet 버전도 제공 (데이터 영구 저장 필요 시 사용)

### 메시지 큐

#### Apache Kafka
- **이미지**: `confluentinc/cp-kafka:7.4.0`
- **포트**: 9092
- **Zookeeper**: `service-zookeeper:2181`
- **토픽 자동 생성**: 활성화
- **리소스**:
  - Requests: 256Mi RAM, 100m CPU
  - Limits: 512Mi RAM, 500m CPU

#### Zookeeper
- **이미지**: `confluentinc/cp-zookeeper:7.4.0`
- **포트**: 2181
- **역할**: Kafka 클러스터 코디네이션

#### Kafka UI
- **이미지**: `provectuslabs/kafka-ui:latest`
- **포트**: 8080
- **용도**: Kafka 클러스터 모니터링 및 관리

### 네트워킹

#### Ingress
- **구성 방식**: 각 overlay에서 환경별로 정의
  - **Dev/Prod (Local)**: NGINX Ingress 사용
  - **EKS**: AWS ALB Ingress Controller 사용
- **Base에서 제거**: 환경별 차이가 크므로 overlay에서만 관리
- **설정 위치**:
  - `overlays/dev/ingress.yaml` - 개발 환경용 NGINX Ingress
  - `overlays/prod/ingress.yaml` - 프로덕션 환경용 NGINX Ingress
  - `overlays/eks/ingress.yaml` - EKS 환경용 ALB Ingress

#### NGINX Ingress Controller
- **용도**: 외부 트래픽 라우팅 및 로드밸런싱 (로컬 환경)
- **별도 설치 필요**: Base kustomization에서 주석 처리됨
- **설치 방법**: Helm 또는 별도 매니페스트 적용

#### MetalLB
- **용도**: 로컬/베어메탈 환경에서 LoadBalancer IP 할당
- **별도 관리**: GitOps 저장소에 포함되지 않음
- **설정 파일**: `metallb/ipaddresspool.yaml` (참고용)

---

## 환경 관리

### 환경별 차이점

| 항목 | Dev | Staging | Prod (Local) | EKS (Cloud) |
|------|-----|---------|--------------|-------------|
| **배포 위치** | 로컬 개발 환경 | 로컬 스테이징 | 로컬 PC | AWS EKS |
| **Namespace** | `trip-service-dev` | `trip-service-staging` | `trip-service-prod` | `trip-service-prod` |
| **Replicas** | 1 | 2 | 3 | 3 |
| **Image Registry** | Docker Hub | Docker Hub | Docker Hub | AWS ECR |
| **Image Tag** | `dev-latest` | `staging-latest` | `prod-63` (Jenkins) | `latest` |
| **Resources** | 최소 | 중간 | 높음 (512Mi) | 높음 (512Mi) |
| **Ingress** | NGINX | NGINX | NGINX | AWS ALB |
| **Storage** | local-path | local-path | local-path | AWS EBS (gp3-encrypted) |
| **LoadBalancer** | MetalLB | MetalLB | MetalLB | AWS NLB |
| **도메인** | 로컬 | 로컬 | 로컬 | 2025teamproject.store |
| **SSL/TLS** | ❌ | ❌ | ❌ | ✅ (ACM) |

### Dev 환경

**네임스페이스**: `trip-service-dev`

**특징**:
- 모든 서비스 레플리카 1개로 최소화
- `dev-latest` 태그 사용 (항상 최신 이미지)
- 낮은 리소스 제한
- 빠른 배포 및 테스트에 최적화

**Kustomization**:
```yaml
namespace: trip-service-dev
images:
  - name: korgosu/service-currency
    newTag: dev-latest
replicas:
  - name: service-currency
    count: 1
```

### Prod 환경 (로컬 PC)

**배포 위치**: 로컬 쿠버네티스 클러스터 (Docker Desktop, Minikube, Kind 등)

**네임스페이스**: `trip-service-prod`

**용도**: 로컬 환경에서 프로덕션급 설정 테스트 및 검증

**특징**:
- 모든 서비스 레플리카 3개로 고가용성 확보
- Jenkins CI/CD 파이프라인에서 빌드된 이미지 사용
- 특정 버전 태그 사용: Jenkins 빌드 번호로 지정 (`prod-63`)
- 높은 리소스 제한 (512Mi RAM, 500m CPU)
- Docker Hub 이미지 사용 (`korgosu/*`)
- 환경 변수로 로그 레벨 및 API URL 설정
- NGINX Ingress Controller 사용
- MetalLB로 LoadBalancer 타입 서비스 지원

**주요 패치**:
```yaml
patches:
  # 이미지 버전 지정
  - target:
      kind: Deployment
      name: service-currency
    patch: |-
      - op: replace
        path: /spec/template/spec/containers/0/image
        value: korgosu/service-currency:prod-63
      - op: replace
        path: /spec/replicas
        value: 3
      - op: replace
        path: /spec/template/spec/containers/0/resources
        value:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"

  # 환경 설정
  - target:
      kind: ConfigMap
      name: trip-service-config
    patch: |-
      - op: add
        path: /data/ENVIRONMENT
        value: "prod"
      - op: add
        path: /data/LOG_LEVEL
        value: "info"
```

**사용 시나리오**:
- 클라우드 배포 전 최종 검증
- 프로덕션 환경 시뮬레이션
- 성능 테스트 및 부하 테스트
- 롤링 업데이트 전략 검증

### EKS 환경 (AWS 클라우드)

**배포 위치**: AWS EKS (Elastic Kubernetes Service)

**네임스페이스**: `trip-service-prod`

**용도**: 실제 프로덕션 운영 환경

**특징**:
- **AWS ECR 이미지 레지스트리**: `716773066105.dkr.ecr.ap-northeast-2.amazonaws.com`
- **AWS ALB Ingress Controller**: Application Load Balancer 자동 프로비저닝
- **도메인**: `2025teamproject.store`
- **SSL/TLS**: AWS Certificate Manager (ACM) 인증서 적용
- **리전**: `ap-northeast-2` (서울)
- **스토리지**: AWS EBS gp3 볼륨 (암호화, `gp3-encrypted` StorageClass)
- **로드밸런서**: AWS NLB (Network Load Balancer) for frontend

**주요 패치**:
```yaml
patches:
  # ECR 이미지 사용
  - target:
      kind: Deployment
      name: service-currency
    patch: |-
      - op: replace
        path: /spec/template/spec/containers/0/image
        value: 716773066105.dkr.ecr.ap-northeast-2.amazonaws.com/service-currency:latest

  # 참고: StatefulSet의 volumeClaimTemplates는 base에서 storageClassName: gp3-encrypted로 설정됨
  # PVC는 StatefulSet이 자동 생성하므로 별도 패치 불필요

  # 환경 변수 설정
  - target:
      kind: ConfigMap
      name: trip-service-config
    patch: |-
      - op: add
        path: /data/ENVIRONMENT
        value: "prod"
      - op: add
        path: /data/CLOUD_PROVIDER
        value: "aws"
      - op: add
        path: /data/REGION
        value: "ap-northeast-2"
```

**Ingress 설정**:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: trip-service-ingress
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/load-balancer-name: trip-service-alb
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'
    alb.ingress.kubernetes.io/ssl-redirect: '443'
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:ap-northeast-2:716773066105:certificate/...
spec:
  ingressClassName: alb
  rules:
  - host: 2025teamproject.store
    http:
      paths:
      - path: /api/v1/currencies
        pathType: Prefix
        backend:
          service:
            name: service-currency
            port:
              number: 8000
      - path: /
        pathType: Prefix
        backend:
          service:
            name: service-frontend
            port:
              number: 80
```

**AWS 리소스**:
- **EKS 클러스터**: 관리형 쿠버네티스 클러스터
- **ECR**: 컨테이너 이미지 저장소
- **ALB**: Layer 7 로드밸런서 (HTTP/HTTPS)
- **NLB**: Layer 4 로드밸런서 (Frontend용)
- **EBS**: 영구 볼륨 스토리지
- **ACM**: SSL/TLS 인증서 관리
- **Route 53**: DNS 관리 (선택사항)

---

## 배포 가이드

### 사전 요구사항

1. **쿠버네티스 클러스터** (v1.24+)
2. **kubectl** CLI 도구
3. **kustomize** (kubectl에 내장됨)
4. **Ingress Controller** (NGINX 또는 ALB)
5. **Secrets 설정** (MySQL, MongoDB 비밀번호 등)

### 로컬 환경 배포

#### 1. Secrets 생성

```bash
kubectl create namespace trip-service-dev

kubectl create secret generic trip-service-secrets \
  --from-literal=mysql-password=your-mysql-password \
  --from-literal=mongodb-password=your-mongodb-password \
  --from-literal=redis-password=your-redis-password \
  -n trip-service-dev
```

#### 2. Kustomize로 배포

```bash
# Dev 환경 배포
kubectl apply -k k8s/overlays/dev

# 배포 확인
kubectl get all -n trip-service-dev
```

#### 3. Ingress 설정 확인

```bash
# Ingress 상태 확인
kubectl get ingress -n trip-service-dev

# Ingress Controller 로그 확인
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller
```

### ArgoCD를 통한 GitOps 배포

#### 1. ArgoCD Application 생성

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: trip-service-dev
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/your-org/trip-currency-local-gitops
    targetRevision: main
    path: k8s/overlays/dev
  destination:
    server: https://kubernetes.default.svc
    namespace: trip-service-dev
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

#### 2. ArgoCD CLI로 배포

```bash
argocd app create trip-service-dev \
  --repo https://github.com/your-org/trip-currency-local-gitops \
  --path k8s/overlays/dev \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace trip-service-dev \
  --sync-policy automated
```

#### 3. 배포 상태 확인

```bash
# ArgoCD UI에서 확인
argocd app get trip-service-dev

# 동기화 실행
argocd app sync trip-service-dev
```

### 로컬 프로덕션 배포 (Prod Overlay)

로컬 환경에서 프로덕션급 설정으로 배포합니다.

```bash
# 1. Secrets 생성 (프로덕션 값 사용)
kubectl create namespace trip-service-prod

kubectl create secret generic trip-service-secrets \
  --from-literal=mysql-password=PROD_MYSQL_PASSWORD \
  --from-literal=mongodb-password=PROD_MONGODB_PASSWORD \
  --from-literal=redis-password=PROD_REDIS_PASSWORD \
  -n trip-service-prod

# 2. Prod 오버레이로 배포
kubectl apply -k k8s/overlays/prod

# 3. 배포 확인
kubectl get all -n trip-service-prod

# 4. 롤링 업데이트 모니터링
kubectl rollout status deployment/service-currency -n trip-service-prod
kubectl rollout status deployment/service-history -n trip-service-prod
kubectl rollout status deployment/service-ranking -n trip-service-prod
kubectl rollout status deployment/service-frontend -n trip-service-prod

# 5. Pod 상태 확인
kubectl get pods -n trip-service-prod -o wide

# 6. Ingress 확인
kubectl get ingress -n trip-service-prod
```

### AWS EKS 배포 (EKS Overlay)

실제 AWS 클라우드 환경에 배포합니다.

#### 사전 요구사항

1. **AWS CLI 설치 및 설정**
```bash
aws configure
# AWS Access Key ID, Secret Access Key, Region (ap-northeast-2) 설정
```

2. **EKS 클러스터 생성** (이미 생성되어 있다고 가정)
```bash
# eksctl로 클러스터 생성 (예시)
eksctl create cluster \
  --name trip-service-cluster \
  --region ap-northeast-2 \
  --nodegroup-name standard-workers \
  --node-type t3.medium \
  --nodes 3 \
  --nodes-min 1 \
  --nodes-max 4
```

3. **kubectl 컨텍스트 설정**
```bash
aws eks update-kubeconfig --name trip-service-cluster --region ap-northeast-2
```

4. **AWS Load Balancer Controller 설치**
```bash
# IAM 정책 생성
curl -o iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.4.0/docs/install/iam_policy.json

aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam-policy.json

# Helm으로 AWS Load Balancer Controller 설치
helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=trip-service-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

5. **ECR 인증 설정**
```bash
# ECR 로그인 (이미지 빌드/푸시 시)
aws ecr get-login-password --region ap-northeast-2 | \
  docker login --username AWS --password-stdin 716773066105.dkr.ecr.ap-northeast-2.amazonaws.com
```

#### EKS 배포 절차

```bash
# 1. EKS 클러스터 접속 확인
kubectl cluster-info

# 2. Namespace 생성
kubectl create namespace trip-service-prod

# 3. Secrets 생성
kubectl create secret generic trip-service-secrets \
  --from-literal=mysql-password=PROD_MYSQL_PASSWORD \
  --from-literal=mongodb-password=PROD_MONGODB_PASSWORD \
  --from-literal=redis-password=PROD_REDIS_PASSWORD \
  -n trip-service-prod

# 4. EKS 오버레이로 배포
kubectl apply -k k8s/overlays/eks

# 5. 배포 상태 확인
kubectl get all -n trip-service-prod

# 6. Pod 상태 확인
kubectl get pods -n trip-service-prod -o wide

# 7. ALB Ingress 프로비저닝 확인 (몇 분 소요)
kubectl get ingress -n trip-service-prod
kubectl describe ingress trip-service-ingress -n trip-service-prod

# 8. ALB DNS 이름 확인
kubectl get ingress trip-service-ingress -n trip-service-prod -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'

# 9. 서비스 테스트
# ALB DNS를 도메인(2025teamproject.store)의 CNAME 레코드로 설정 후 테스트
curl https://2025teamproject.store/health
```

#### ArgoCD를 통한 EKS GitOps 배포

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: trip-service-eks-prod
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/your-org/trip-currency-local-gitops
    targetRevision: main
    path: k8s/overlays/eks
  destination:
    server: https://YOUR-EKS-CLUSTER-ENDPOINT
    namespace: trip-service-prod
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

#### EKS 배포 검증

```bash
# 1. 모든 Pod가 Running 상태인지 확인
kubectl get pods -n trip-service-prod

# 2. PVC가 EBS 볼륨에 바인딩되었는지 확인 (StatefulSet이 자동 생성)
kubectl get pvc -n trip-service-prod
# 예상 PVC: mysql-storage-mysql-0, mongodb-storage-mongodb-0
# StorageClass 확인: kubectl get pvc -n trip-service-prod -o jsonpath='{.items[*].spec.storageClassName}'

# 3. Ingress ALB 생성 확인
kubectl get ingress -n trip-service-prod

# 4. 서비스 엔드포인트 테스트
curl https://2025teamproject.store/api/v1/currencies
curl https://2025teamproject.store/api/v1/history
curl https://2025teamproject.store/api/v1/rankings

# 5. SSL/TLS 인증서 확인
curl -vI https://2025teamproject.store

# 6. 로그 확인
kubectl logs -f deployment/service-currency -n trip-service-prod
```

#### EKS 리소스 모니터링

```bash
# Node 리소스 사용량
kubectl top nodes

# Pod 리소스 사용량
kubectl top pods -n trip-service-prod

# HPA 상태 확인
kubectl get hpa -n trip-service-prod

# AWS 콘솔에서 확인
# - EKS 클러스터 상태
# - EC2 인스턴스 (Worker Nodes)
# - ALB 대상 그룹 상태
# - EBS 볼륨
# - CloudWatch 로그 및 메트릭
```

#### 비용 최적화 팁

1. **적절한 인스턴스 타입 선택**: t3.medium 또는 t3.small 사용
2. **Auto Scaling 설정**: HPA로 필요 시에만 Pod 증가
3. **스팟 인스턴스 활용**: 비용 절감 (최대 90%)
4. **불필요한 리소스 정리**: 미사용 EBS 볼륨, ALB 등 삭제
5. **Reserved Instances**: 장기 운영 시 예약 인스턴스 구매

---

## 모니터링 및 관찰성

### Prometheus 메트릭

모든 애플리케이션 서비스는 Prometheus 스크래핑을 위한 어노테이션 포함:

```yaml
annotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "8000"
  prometheus.io/path: "/metrics"
```

### 헬스체크

- **Liveness Probe**: 컨테이너가 살아있는지 확인
- **Readiness Probe**: 트래픽을 받을 준비가 되었는지 확인
- **엔드포인트**: `/health`

### 로깅

- **stdout/stderr**: 컨테이너 로그는 표준 출력으로 전송
- **로그 수집**: Fluentd, Fluent Bit 등을 통해 중앙 집중식 로깅 가능

### 주요 모니터링 지표

#### 애플리케이션 메트릭
- HTTP 요청 수 및 레이턴시
- 에러율
- 데이터베이스 쿼리 성능

#### 인프라 메트릭
- CPU 사용률
- 메모리 사용률
- 네트워크 I/O
- 디스크 I/O

---

## 리소스 관리

### CPU 및 메모리 요청/제한

#### 개발 환경

| 서비스 | CPU Request | CPU Limit | Memory Request | Memory Limit |
|--------|-------------|-----------|----------------|--------------|
| Currency | 100m | 500m | 256Mi | 384Mi |
| History | 100m | 500m | 256Mi | 384Mi |
| Ranking | 100m | 500m | 256Mi | 384Mi |
| Frontend | 50m | 200m | 32Mi | 64Mi |
| DataIngestor | 100m | 500m | 256Mi | 512Mi |

#### 프로덕션 환경

| 서비스 | CPU Request | CPU Limit | Memory Request | Memory Limit |
|--------|-------------|-----------|----------------|--------------|
| Currency | 100m | 500m | 256Mi | 512Mi |
| History | 100m | 500m | 256Mi | 512Mi |
| Ranking | 100m | 500m | 256Mi | 512Mi |
| Frontend | 100m | 500m | 256Mi | 512Mi |
| DataIngestor | 100m | 500m | 256Mi | 512Mi |

### HPA (Horizontal Pod Autoscaler)

#### 설정

```yaml
minReplicas: 1
maxReplicas: 10
metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 90
```

#### 동작 방식

- **CPU 70% 초과 시**: 스케일 아웃
- **Memory 90% 초과 시**: 스케일 아웃
- **부하 감소 시**: 자동으로 스케일 인 (cooldown 기간 후)

### 스토리지

#### Persistent Volume Claims

| 리소스 | 스토리지 크기 | Access Mode | 생성 방식 |
|--------|--------------|-------------|----------|
| MySQL | 10Gi | ReadWriteOnce | StatefulSet `volumeClaimTemplates` 자동 생성 |
| MongoDB | 10Gi | ReadWriteOnce | StatefulSet `volumeClaimTemplates` 자동 생성 |
| Redis | 5Gi | ReadWriteOnce | StatefulSet `volumeClaimTemplates` (예비용, 현재는 Deployment 사용) |

#### StorageClass

- **로컬/개발**: 기본 StorageClass (hostPath 또는 local-path-provisioner)
- **EKS**: `gp3-encrypted` (AWS EBS gp3, 암호화 활성화)
  - StatefulSet의 `volumeClaimTemplates`에서 자동으로 `gp3-encrypted` 사용
  - 암호화된 EBS 볼륨으로 데이터 보안 강화
  - 볼륨 확장 가능 (`allowVolumeExpansion: true`)

#### StatefulSet 스토리지 관리

- **자동 PVC 생성**: StatefulSet이 각 Pod에 대해 고유한 PVC를 자동 생성
  - MySQL: `mysql-storage-mysql-0`
  - MongoDB: `mongodb-storage-mongodb-0`
- **데이터 영구성**: Pod 재시작 시에도 데이터 유지
- **안정적인 네트워크**: Pod 이름이 고정되어 안정적인 DNS 이름 제공

---

## 문제 해결

### 일반적인 문제

#### 1. Pod가 시작되지 않음

```bash
# Pod 상태 확인
kubectl get pods -n trip-service-dev

# Pod 로그 확인
kubectl logs <pod-name> -n trip-service-dev

# Pod 이벤트 확인
kubectl describe pod <pod-name> -n trip-service-dev
```

#### 2. 데이터베이스 연결 실패

- Secrets가 올바르게 생성되었는지 확인
- StatefulSet Pod가 Running 상태인지 확인 (`kubectl get statefulset -n <namespace>`)
- Headless Service가 올바르게 설정되었는지 확인 (`kubectl get svc service-mysql service-mongodb`)
- PVC가 바인딩되었는지 확인 (`kubectl get pvc -n <namespace>`)
- 네트워크 정책이 통신을 차단하지 않는지 확인
- Service 이름으로 접근 가능한지 확인 (Headless Service도 Service 이름으로 접근 가능)

#### 3. Ingress 404 에러

```bash
# Ingress 규칙 확인
kubectl get ingress -n trip-service-dev -o yaml

# Ingress Controller 로그 확인
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller
```

#### 4. HPA가 작동하지 않음

```bash
# Metrics Server 설치 확인
kubectl get deployment metrics-server -n kube-system

# HPA 상태 확인
kubectl get hpa -n trip-service-dev
kubectl describe hpa <hpa-name> -n trip-service-dev
```

---

## 기여 가이드

### 변경 사항 적용 절차

1. **브랜치 생성**: `git checkout -b feature/my-changes`
2. **매니페스트 수정**: 필요한 YAML 파일 편집
3. **로컬 테스트**: `kubectl apply -k k8s/overlays/dev --dry-run=client`
4. **커밋 및 푸시**: `git commit -m "Add: new feature" && git push`
5. **Pull Request 생성**: GitHub에서 PR 생성
6. **리뷰 및 머지**: 팀 리뷰 후 main 브랜치로 머지
7. **자동 배포**: ArgoCD가 변경 사항 감지 및 배포

### 베스트 프랙티스

- **작은 단위로 커밋**: 한 번에 하나의 변경 사항만 포함
- **명확한 커밋 메시지**: 변경 이유와 내용을 명확히 기술
- **테스트 후 배포**: 개발 환경에서 충분히 테스트 후 프로덕션 배포
- **Secrets 관리**: 민감한 정보는 절대 Git에 커밋하지 않음
- **버전 태깅**: 프로덕션 배포 시 특정 이미지 태그 사용

---

## 라이선스

MIT License

---

## 문의

문제가 발생하거나 질문이 있으시면 이슈를 생성해주세요.

**관리자**: korgosu
