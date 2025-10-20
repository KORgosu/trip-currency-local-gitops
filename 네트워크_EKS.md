# EKS 네트워크 구조 문서

## 목차
1. [개요](#개요)
2. [네트워크 아키텍처](#네트워크-아키텍처)
3. [VPC 및 서브넷 구성](#vpc-및-서브넷-구성)
4. [Route 53 DNS 구성](#route-53-dns-구성)
5. [로드 밸런서](#로드-밸런서)
6. [보안 그룹](#보안-그룹)
7. [타겟 그룹](#타겟-그룹)
8. [네트워크 트래픽 흐름](#네트워크-트래픽-흐름)
9. [보안 구성](#보안-구성)

---

## 개요

### 클러스터 정보
- **클러스터 이름**: trip-service-cluster
- **리전**: ap-northeast-2 (서울)
- **Kubernetes 버전**: 1.32
- **생성 도구**: eksctl 0.215.0
- **생성일**: 2025-10-13
- **엔드포인트**: https://50F0DAF2BBE0F6806BAB0EC521D87702.gr7.ap-northeast-2.eks.amazonaws.com

### 도메인
- **메인 도메인**: 2025teamproject.store
- **서브 도메인**:
  - grafana.2025teamproject.store (모니터링)
  - jenkins.2025teamproject.store (CI/CD)

---

## 네트워크 아키텍처

```
Internet
    │
    ├─ Route 53 (2025teamproject.store)
    │
    ├─────────────────────────────────────────────────────────────┐
    │                                                               │
    ▼                                                               ▼
[ALB: trip-service-alb]                            [ALB: prometheus-grafana]
    │                                                               │
    │ HTTPS:443 (리스너 규칙)                                       │ HTTPS:443
    ├─ /api/v1/currencies/* → Currency Service                     └─ /* → Grafana (3000)
    ├─ /api/v1/rankings/* → Ranking Service
    ├─ /api/v1/history/* → History Service
    └─ /* → Frontend Service (80)

[NLB: k8s-tripserv-servicef]
    │
    └─ TCP:80 → NodePort:30839 (3 EC2 instances)

    ┌──────────────────────────────────────────────────────────┐
    │                 VPC (192.168.0.0/16)                      │
    │                                                           │
    │  ┌─────────────────────────────────────────────────┐    │
    │  │  Public Subnets                                  │    │
    │  │  - 192.168.0.0/19    (ap-northeast-2d)          │    │
    │  │  - 192.168.32.0/19   (ap-northeast-2a)          │    │
    │  │  - 192.168.64.0/19   (ap-northeast-2c)          │    │
    │  │                                                  │    │
    │  │  [ALB/NLB 배치]                                  │    │
    │  └─────────────────────────────────────────────────┘    │
    │                                                           │
    │  ┌─────────────────────────────────────────────────┐    │
    │  │  Private Subnets                                 │    │
    │  │  - 192.168.96.0/19   (ap-northeast-2d)          │    │
    │  │  - 192.168.128.0/19  (ap-northeast-2a)          │    │
    │  │  - 192.168.160.0/19  (ap-northeast-2c)          │    │
    │  │                                                  │    │
    │  │  [EKS Worker Nodes & Pods]                      │    │
    │  │                                                  │    │
    │  │  ┌───────────────────────────────────┐          │    │
    │  │  │  Pod IPs (192.168.x.x)            │          │    │
    │  │  │  - Currency Service Pods (7개)    │          │    │
    │  │  │  - Ranking Service Pods (3개)     │          │    │
    │  │  │  - History Service Pods (6개)     │          │    │
    │  │  │  - Frontend Pod (1개)             │          │    │
    │  │  │  - Grafana Pod (1개)              │          │    │
    │  │  └───────────────────────────────────┘          │    │
    │  └─────────────────────────────────────────────────┘    │
    │                                                           │
    └──────────────────────────────────────────────────────────┘
```

---

## VPC 및 서브넷 구성

### VPC
- **VPC ID**: vpc-09787158e8b6b2dca
- **CIDR 블록**: 192.168.0.0/16
- **총 IP 주소**: 65,536개

### 퍼블릭 서브넷 (ALB/NLB 배치)

| 서브넷 ID | CIDR | AZ | 용도 | 사용 가능 IP |
|-----------|------|----|----|-------------|
| subnet-0ac8af367a8c2f863 | 192.168.0.0/19 | ap-northeast-2d | 퍼블릭 | 8,134 |
| subnet-0a60c17cc0939fb56 | 192.168.32.0/19 | ap-northeast-2a | 퍼블릭 | 8,151 |
| subnet-098235b751ae8316a | 192.168.64.0/19 | ap-northeast-2c | 퍼블릭 | 8,151 |

**특징**:
- `MapPublicIpOnLaunch: true` (자동 퍼블릭 IP 할당)
- 태그: `kubernetes.io/role/elb: 1` (ELB 배치용)

### 프라이빗 서브넷 (EKS 워커 노드 및 Pod 배치)

| 서브넷 ID | CIDR | AZ | 용도 | 사용 가능 IP |
|-----------|------|----|----|-------------|
| subnet-09e95cfd32ae85443 | 192.168.96.0/19 | ap-northeast-2d | 프라이빗 | 8,184 |
| subnet-049b5be283190c06b | 192.168.128.0/19 | ap-northeast-2a | 프라이빗 | 8,182 |
| subnet-0b29f8571dcb7afc5 | 192.168.160.0/19 | ap-northeast-2c | 프라이빗 | 8,186 |

**특징**:
- `MapPublicIpOnLaunch: false`
- 태그: `kubernetes.io/role/internal-elb: 1` (내부 ELB용)
- EKS 워커 노드와 Pod가 배치됨

---

## Route 53 DNS 구성

### 호스팅 영역
- **도메인**: 2025teamproject.store
- **호스팅 영역 ID**: Z03438002JV848JX60ZRK
- **레코드 개수**: 7개

### DNS 레코드

#### 1. 메인 도메인 (2025teamproject.store)
```
타입: A (Alias)
대상: trip-service-alb-1833853365.ap-northeast-2.elb.amazonaws.com
라우팅 정책: Weighted (가중치: 100)
식별자: EKS-Cluster
헬스 체크: 50cf0418-a8eb-47cd-a8e2-4ac64c2d221a
```

#### 2. Grafana 서브도메인
```
타입: A (Alias)
도메인: grafana.2025teamproject.store
대상: k8s-monitori-promethe-4dcb6c3ce9-603147373.ap-northeast-2.elb.amazonaws.com
헬스 체크: 활성화됨
```

#### 3. Jenkins 서브도메인
```
타입: A (Alias)
도메인: jenkins.2025teamproject.store
대상: dualstack.jenkins-alb-290859379.ap-northeast-2.elb.amazonaws.com
헬스 체크: 활성화됨
```

#### 4. SSL 인증서 검증 레코드
- ACM 인증서 검증을 위한 CNAME 레코드 2개 존재

---

## 로드 밸런서

### 1. trip-service-alb (Application Load Balancer)

#### 기본 정보
- **ARN**: arn:aws:elasticloadbalancing:ap-northeast-2:716773066105:loadbalancer/app/trip-service-alb/35fa0b2d3b66cb32
- **DNS**: trip-service-alb-1833853365.ap-northeast-2.elb.amazonaws.com
- **타입**: Application Load Balancer
- **Scheme**: internet-facing
- **생성일**: 2025-10-13
- **가용 영역**: 2a, 2c, 2d

#### 보안 그룹
- sg-034f61e0a791bada2 (Managed LB SecurityGroup)
- sg-0bf801052c9c22014 (Shared Backend SecurityGroup)

#### 리스너

##### HTTP 리스너 (포트 80)
- **동작**: HTTPS로 리다이렉트 (301)
- **대상 포트**: 443

##### HTTPS 리스너 (포트 443)
- **SSL 인증서**: arn:aws:acm:ap-northeast-2:716773066105:certificate/feb966da-b32f-48fb-947a-a48c1a077cfa
- **SSL 정책**: ELBSecurityPolicy-2016-08
- **기본 동작**: 404 응답

#### 라우팅 규칙 (우선순위 순)

| 우선순위 | 호스트 헤더 | 경로 패턴 | 타겟 그룹 | 서비스 |
|---------|-----------|---------|----------|--------|
| 1 | 2025teamproject.store | /api/v1/currencies* | k8s-tripserv-servicec-c81086e9b1 | Currency Service (8000) |
| 2 | 2025teamproject.store | /api/v1/rankings* | k8s-tripserv-servicer-cc2fb486a0 | Ranking Service (8000) |
| 3 | 2025teamproject.store | /api/v1/history* | k8s-tripserv-serviceh-9dbd25740e | History Service (8000) |
| 4 | 2025teamproject.store | /health* | k8s-tripserv-servicec-c81086e9b1 | Health Check (8000) |
| 5 | 2025teamproject.store | /* | k8s-tripserv-servicef-b3c3abeb59 | Frontend (80) |
| default | - | - | 404 응답 | - |

### 2. k8s-monitori-promethe ALB (Prometheus/Grafana)

#### 기본 정보
- **ARN**: arn:aws:elasticloadbalancing:ap-northeast-2:716773066105:loadbalancer/app/k8s-monitori-promethe-4dcb6c3ce9/efffcfa216330b4c
- **DNS**: k8s-monitori-promethe-4dcb6c3ce9-603147373.ap-northeast-2.elb.amazonaws.com
- **타입**: Application Load Balancer
- **Scheme**: internet-facing
- **생성일**: 2025-10-14
- **가용 영역**: 2a, 2c, 2d

#### 보안 그룹
- sg-089de9b761b99c72a (Managed LB SecurityGroup)
- sg-0bf801052c9c22014 (Shared Backend SecurityGroup)

#### 리스너

##### HTTP 리스너 (포트 80)
- **동작**: HTTPS로 리다이렉트 (301)

##### HTTPS 리스너 (포트 443)
- **SSL 인증서**: 동일 인증서 사용
- **SSL 정책**: ELBSecurityPolicy-2016-08

#### 라우팅 규칙

| 우선순위 | 호스트 헤더 | 경로 패턴 | 타겟 그룹 | 서비스 |
|---------|-----------|---------|----------|--------|
| 1 | grafana.2025teamproject.store | /* | k8s-monitori-promethe-2260d9f023 | Grafana (3000) |
| default | - | - | 404 응답 | - |

### 3. k8s-tripserv-servicef NLB (Frontend)

#### 기본 정보
- **ARN**: arn:aws:elasticloadbalancing:ap-northeast-2:716773066105:loadbalancer/net/k8s-tripserv-servicef-54ee1d4057/727230bd66326440
- **DNS**: k8s-tripserv-servicef-54ee1d4057-727230bd66326440.elb.ap-northeast-2.amazonaws.com
- **타입**: Network Load Balancer
- **Scheme**: internet-facing
- **생성일**: 2025-10-13
- **가용 영역**: 2a, 2c, 2d

#### 보안 그룹
- sg-0bf801052c9c22014 (Shared Backend SecurityGroup)
- sg-0c1508fbed215db21 (Managed LB SecurityGroup)

#### 리스너

##### TCP 리스너 (포트 80)
- **프로토콜**: TCP
- **타겟 그룹**: k8s-tripserv-servicef-349dffcdbd
- **타겟 포트**: 30839 (NodePort)
- **타겟 타입**: instance (EC2 인스턴스)

---

## 보안 그룹

### EKS 클러스터 핵심 보안 그룹

#### 1. sg-0fd9cbbe5ade917d2 - ControlPlaneSecurityGroup
**목적**: Control Plane과 Worker Node 간 통신 관리

**인바운드 규칙**: 없음

**아웃바운드 규칙**:
- 모든 프로토콜 → 0.0.0.0/0 (전체 허용)

#### 2. sg-066ff3c2a7e86298f - EKS Cluster SecurityGroup
**목적**: EKS Control Plane ENI 및 관리형 워크로드에 적용

**인바운드 규칙**:
1. 모든 프로토콜 ← sg-066ff3c2a7e86298f (자기 자신)
2. 모든 프로토콜 ← sg-024fd1f24408140ca (워커 노드)
3. TCP 80-30839 ← sg-0bf801052c9c22014 (백엔드 SG)

**아웃바운드 규칙**:
- 모든 프로토콜 → 0.0.0.0/0

#### 3. sg-024fd1f24408140ca - ClusterSharedNodeSecurityGroup
**목적**: 클러스터 내 모든 노드 간 통신

**인바운드 규칙**:
1. 모든 프로토콜 ← sg-024fd1f24408140ca (노드 간 통신)
2. 모든 프로토콜 ← sg-066ff3c2a7e86298f (클러스터 SG)

**아웃바운드 규칙**:
- 모든 프로토콜 → 0.0.0.0/0

### 로드 밸런서 보안 그룹

#### 4. sg-0bf801052c9c22014 - Shared Backend SecurityGroup
**목적**: 모든 로드 밸런서에서 공유하는 백엔드 보안 그룹

**태그**: `elbv2.k8s.aws/resource: backend-sg`

**인바운드 규칙**: 없음

**아웃바운드 규칙**:
- 모든 프로토콜 → 0.0.0.0/0

**연결**: 모든 ALB 및 NLB에 연결

#### 5. sg-034f61e0a791bada2 - trip-service-ingress ALB SecurityGroup
**목적**: trip-service-alb의 인터넷 트래픽 수신

**인바운드 규칙**:
1. TCP 80 ← 0.0.0.0/0 (HTTP)
2. TCP 443 ← 0.0.0.0/0 (HTTPS)

**아웃바운드 규칙**:
- 모든 프로토콜 → 0.0.0.0/0

**연결**: trip-service-alb

#### 6. sg-089de9b761b99c72a - prometheus-grafana ALB SecurityGroup
**목적**: 모니터링 시스템 접근 제어

**인바운드 규칙**:
1. TCP 80 ← 0.0.0.0/0 (HTTP)
2. TCP 443 ← 0.0.0.0/0 (HTTPS)

**아웃바운드 규칙**:
- 모든 프로토콜 → 0.0.0.0/0

**연결**: k8s-monitori-promethe ALB

#### 7. sg-0c1508fbed215db21 - service-frontend NLB SecurityGroup
**목적**: 프론트엔드 서비스 NLB 트래픽 제어

**인바운드 규칙**:
1. TCP 80 ← 0.0.0.0/0 (HTTP)

**아웃바운드 규칙**:
- 모든 프로토콜 → 0.0.0.0/0

**연결**: k8s-tripserv-servicef NLB

### 보안 그룹 관계도

```
[Internet: 0.0.0.0/0]
        │
        ├─ TCP 80, 443 → [sg-034f61e0a791bada2] → trip-service-alb
        ├─ TCP 80, 443 → [sg-089de9b761b99c72a] → prometheus-grafana ALB
        └─ TCP 80      → [sg-0c1508fbed215db21] → frontend NLB
                               │
                               ├─ [sg-0bf801052c9c22014] (Shared Backend)
                               │         │
                               │         └─ TCP 80-30839 → [sg-066ff3c2a7e86298f] (Cluster SG)
                               │                                    │
                               │                                    ├─ [sg-066ff3c2a7e86298f] (자체)
                               │                                    └─ [sg-024fd1f24408140ca] (Node SG)
                               │                                              │
                               └───────────────────────────────────────────────┘
```

---

## 타겟 그룹

### 1. trip-service-alb 연결 타겟 그룹

#### k8s-tripserv-servicef-b3c3abeb59 (Frontend)
- **프로토콜**: HTTP
- **포트**: 80
- **타겟 타입**: IP (Pod 직접 연결)
- **헬스체크**: /health (15초 간격)
- **타겟 상태**: 1개 healthy
  - 192.168.62.137:80

#### k8s-tripserv-servicec-c81086e9b1 (Currency Service)
- **프로토콜**: HTTP
- **포트**: 8000
- **타겟 타입**: IP
- **헬스체크**: /health (15초 간격)
- **타겟 상태**: 7개 healthy
  - 192.168.62.140:8000 (2a)
  - 192.168.62.142:8000 (2a)
  - 192.168.37.98:8000 (2a)
  - 192.168.75.225:8000 (2c)
  - 192.168.2.208:8000 (2d)
  - 192.168.16.157:8000 (2d)
  - 192.168.82.19:8000 (2c)

#### k8s-tripserv-servicer-cc2fb486a0 (Ranking Service)
- **프로토콜**: HTTP
- **포트**: 8000
- **타겟 타입**: IP
- **헬스체크**: /health (15초 간격)
- **타겟 상태**: 3개 healthy
  - 192.168.15.32:8000 (2d)
  - 192.168.82.21:8000 (2c)
  - 192.168.62.139:8000 (2a)

#### k8s-tripserv-serviceh-9dbd25740e (History Service)
- **프로토콜**: HTTP
- **포트**: 8000
- **타겟 타입**: IP
- **헬스체크**: /health (15초 간격)
- **타겟 상태**: 6개 healthy
  - 192.168.75.226:8000 (2c)
  - 192.168.75.224:8000 (2c)
  - 192.168.16.159:8000 (2d)
  - 192.168.62.141:8000 (2a)
  - 192.168.62.143:8000 (2a)
  - 192.168.16.158:8000 (2d)

### 2. NLB 연결 타겟 그룹

#### k8s-tripserv-servicef-349dffcdbd (Frontend NodePort)
- **프로토콜**: TCP
- **포트**: 30839 (NodePort)
- **타겟 타입**: instance (EC2 인스턴스)
- **헬스체크**: TCP (10초 간격)
- **로드 밸런서**: k8s-tripserv-servicef NLB
- **타겟 상태**: 3개 healthy
  - i-0a0f682f70f7266b3:30839
  - i-0bcf4ef147f2621eb:30839
  - i-0ed63cafb4dad0f3f:30839

### 3. 모니터링용 타겟 그룹

#### k8s-monitori-promethe-2260d9f023 (Grafana)
- **프로토콜**: HTTP
- **포트**: 3000
- **타겟 타입**: IP
- **헬스체크**: /api/health (15초 간격)
- **로드 밸런서**: prometheus-grafana ALB
- **타겟 상태**: 1개 healthy
  - 192.168.62.130:3000 (2a)

### 4. 미사용 타겟 그룹

#### jenkins-master-tg
- **프로토콜**: HTTP
- **포트**: 8080
- **VPC**: vpc-03efe133e68a106a1 (다른 VPC)
- **로드 밸런서**: 없음 (미연결)
- **상태**: 사용되지 않음

---

## 네트워크 트래픽 흐름

### 1. 메인 웹 애플리케이션 트래픽

#### Currency API 요청 흐름
```
사용자
  │
  ├─ DNS 조회: 2025teamproject.store
  │   └─ Route 53 → trip-service-alb DNS
  │
  ├─ HTTPS 요청: https://2025teamproject.store/api/v1/currencies
  │
  ├─ [Internet Gateway]
  │
  ├─ [trip-service-alb] (Public Subnet)
  │   │
  │   ├─ 보안그룹 체크 (sg-034f61e0a791bada2)
  │   │   └─ TCP 443 허용됨
  │   │
  │   ├─ HTTP 리스너 (포트 80)
  │   │   └─ 301 Redirect → HTTPS
  │   │
  │   ├─ HTTPS 리스너 (포트 443)
  │   │   ├─ SSL 종료 (ACM 인증서)
  │   │   └─ 호스트/경로 기반 라우팅
  │   │
  │   └─ 라우팅 규칙 #1 매칭
  │       └─ Host: 2025teamproject.store
  │       └─ Path: /api/v1/currencies*
  │
  ├─ [타겟그룹: k8s-tripserv-servicec-c81086e9b1]
  │   │
  │   ├─ 헬스체크: /health (15초마다)
  │   │
  │   └─ 로드 밸런싱 (7개 타겟)
  │
  ├─ [보안그룹 체크]
  │   ├─ sg-0bf801052c9c22014 (Backend SG)
  │   └─ sg-066ff3c2a7e86298f (Cluster SG)
  │       └─ TCP 80-30839 허용
  │
  ├─ [Private Subnet - Pod IP]
  │   │
  │   └─ Currency Service Pod (192.168.x.x:8000)
  │       └─ 3개 AZ에 분산 배치
  │
  └─ 응답 (역순)
```

#### Frontend 요청 흐름 (NLB 사용)
```
사용자
  │
  ├─ HTTP 요청: http://[NLB-DNS]/
  │
  ├─ [k8s-tripserv-servicef NLB] (Public Subnet)
  │   │
  │   ├─ 보안그룹 체크 (sg-0c1508fbed215db21)
  │   │   └─ TCP 80 허용됨
  │   │
  │   └─ TCP 리스너 (포트 80)
  │
  ├─ [타겟그룹: k8s-tripserv-servicef-349dffcdbd]
  │   │
  │   ├─ 타겟 타입: EC2 Instance
  │   ├─ NodePort: 30839
  │   └─ 헬스체크: TCP (10초마다)
  │
  ├─ [EC2 Worker Node] (3개 노드에 로드 밸런싱)
  │   │
  │   └─ NodePort 30839
  │
  ├─ [Kubernetes Service]
  │   │
  │   └─ ClusterIP → Pod 매핑
  │
  └─ [Frontend Pod] (192.168.62.137:80)
```

#### Frontend 요청 흐름 (ALB 사용)
```
사용자
  │
  ├─ HTTPS 요청: https://2025teamproject.store/
  │
  ├─ [trip-service-alb]
  │   │
  │   └─ 라우팅 규칙 #5 매칭
  │       └─ Host: 2025teamproject.store
  │       └─ Path: /*
  │
  ├─ [타겟그룹: k8s-tripserv-servicef-b3c3abeb59]
  │   │
  │   └─ 타겟 타입: IP (Pod 직접)
  │
  └─ [Frontend Pod] (192.168.62.137:80)
```

### 2. 모니터링 트래픽 (Grafana)

```
사용자
  │
  ├─ DNS 조회: grafana.2025teamproject.store
  │   └─ Route 53 → prometheus-grafana ALB DNS
  │
  ├─ HTTPS 요청: https://grafana.2025teamproject.store/
  │
  ├─ [k8s-monitori-promethe ALB] (Public Subnet)
  │   │
  │   ├─ 보안그룹 체크 (sg-089de9b761b99c72a)
  │   │   └─ TCP 443 허용됨
  │   │
  │   ├─ HTTP 리스너 (포트 80)
  │   │   └─ 301 Redirect → HTTPS
  │   │
  │   ├─ HTTPS 리스너 (포트 443)
  │   │   ├─ SSL 종료
  │   │   └─ 호스트 기반 라우팅
  │   │
  │   └─ 라우팅 규칙 #1 매칭
  │       └─ Host: grafana.2025teamproject.store
  │       └─ Path: /*
  │
  ├─ [타겟그룹: k8s-monitori-promethe-2260d9f023]
  │   │
  │   ├─ 헬스체크: /api/health
  │   └─ 타겟: 1개
  │
  └─ [Grafana Pod] (192.168.62.130:3000)
```

### 3. 서비스 간 내부 통신

```
[Pod A] (192.168.x.x)
  │
  ├─ Kubernetes Service DNS
  │   └─ 예: currency-service.trip-service-prod.svc.cluster.local
  │
  ├─ [Kubernetes Service ClusterIP]
  │   │
  │   └─ kube-proxy (iptables/IPVS 규칙)
  │
  ├─ 보안그룹 체크
  │   ├─ sg-066ff3c2a7e86298f (Cluster SG)
  │   └─ sg-024fd1f24408140ca (Node SG)
  │       └─ 자기 자신 간 모든 트래픽 허용
  │
  └─ [Pod B] (192.168.y.y)
```

---

## 보안 구성

### 1. 계층별 보안

#### Layer 7 (Application Layer)
- **ALB**: 호스트 헤더 및 경로 패턴 기반 라우팅
- **SSL/TLS**: ACM 인증서로 HTTPS 암호화
- **SSL 정책**: ELBSecurityPolicy-2016-08
- **HTTP → HTTPS 리다이렉트**: 모든 HTTP 트래픽 강제 HTTPS 전환

#### Layer 4 (Transport Layer)
- **NLB**: TCP 로드 밸런싱
- **보안 그룹**: 포트 기반 접근 제어

#### Layer 3 (Network Layer)
- **VPC 격리**: 192.168.0.0/16 프라이빗 네트워크
- **서브넷 분리**: Public/Private 서브넷 분리
- **보안 그룹**: Stateful 방화벽

### 2. 보안 그룹 전략

#### 인터넷 접근 제어
```
Internet (0.0.0.0/0)
    │
    └─ TCP 80, 443만 허용
        │
        └─ ALB/NLB 보안 그룹
            │
            └─ 백엔드 보안 그룹
                │
                └─ 클러스터/노드 보안 그룹
                    │
                    └─ Pod 통신
```

#### 최소 권한 원칙
1. **ALB SG**: 인터넷에서 80, 443만 허용
2. **Backend SG**: ALB에서 백엔드로 트래픽 전달
3. **Cluster SG**: 클러스터 내부 통신만 허용
4. **Node SG**: 노드 간 모든 통신 허용 (Kubernetes 요구사항)

### 3. 네트워크 격리

#### Public Subnet
- **배치 리소스**: ALB, NLB
- **인터넷 게이트웨이**: 직접 연결
- **용도**: 외부 트래픽 수신

#### Private Subnet
- **배치 리소스**: EKS 워커 노드, Pod
- **NAT 게이트웨이**: 아웃바운드 트래픽용
- **용도**: 애플리케이션 실행

### 4. AWS Load Balancer Controller

#### 자동 관리 항목
- 타겟 그룹 생성/삭제
- ALB/NLB 프로비저닝
- 보안 그룹 규칙 자동 업데이트
- Pod IP 자동 등록/해제

#### 주요 기능
1. **IP 타겟 모드**: Pod IP로 직접 라우팅 (성능 향상)
2. **동적 타겟 관리**: Pod 증감에 따라 자동 업데이트
3. **헬스 체크**: Kubernetes Readiness Probe 연동

### 5. 고가용성 (HA) 구성

#### Multi-AZ 배포
- **AZ 개수**: 3개 (2a, 2c, 2d)
- **ALB/NLB**: 모든 AZ에 배치
- **Pod 분산**: 3개 AZ에 균등 분산

#### 타겟 상태
- **Currency Service**: 7개 Pod (2a: 3, 2c: 2, 2d: 2)
- **Ranking Service**: 3개 Pod (2a: 1, 2c: 1, 2d: 1)
- **History Service**: 6개 Pod (2a: 2, 2c: 2, 2d: 2)
- **Frontend**: 1개 Pod (2a: 1)
- **Grafana**: 1개 Pod (2a: 1)

### 6. 헬스 체크

#### ALB 헬스 체크
- **엔드포인트**: /health
- **간격**: 15초
- **타임아웃**: 5초
- **정상 임계값**: 2회 연속 성공
- **비정상 임계값**: 2회 연속 실패

#### NLB 헬스 체크
- **프로토콜**: TCP
- **간격**: 10초
- **타임아웃**: 10초
- **정상 임계값**: 3회 연속 성공
- **비정상 임계값**: 3회 연속 실패

### 7. DNS 및 인증서

#### Route 53
- **헬스 체크**: 메인 도메인에 헬스 체크 적용
- **가중치 라우팅**: EKS 클러스터 가중치 100
- **Alias 레코드**: ALB/NLB와 직접 통합

#### ACM 인증서
- **인증서 ARN**: feb966da-b32f-48fb-947a-a48c1a077cfa
- **적용 범위**:
  - 2025teamproject.store
  - grafana.2025teamproject.store
  - jenkins.2025teamproject.store (다른 ALB)
- **검증 방법**: DNS 검증 (CNAME 레코드)

---

## 주요 특징 및 베스트 프랙티스

### 1. AWS Load Balancer Controller 활용
- Kubernetes Ingress/Service와 AWS ALB/NLB 자동 통합
- Pod IP를 타겟으로 직접 라우팅 (NodePort 오버헤드 제거)
- 동적 타겟 관리로 스케일링 자동화

### 2. 3계층 보안 아키텍처
```
Internet → ALB/NLB SG → Shared Backend SG → Cluster/Node SG → Pod
```

### 3. Multi-AZ 고가용성
- 3개 가용 영역에 리소스 분산
- ALB/NLB의 자동 장애 조치
- Pod 레플리카의 AZ 분산 배치

### 4. SSL/TLS 암호화
- 모든 외부 트래픽 HTTPS 강제
- ACM 관리형 인증서 사용
- HTTP → HTTPS 자동 리다이렉트

### 5. 마이크로서비스 아키텍처
- 경로 기반 라우팅으로 서비스 분리
  - /api/v1/currencies → Currency Service
  - /api/v1/rankings → Ranking Service
  - /api/v1/history → History Service
  - / → Frontend Service

### 6. 모니터링 및 관찰성
- 전용 Grafana 서브도메인
- 별도 ALB로 모니터링 트래픽 격리
- 헬스 체크 엔드포인트 분리

### 7. 네트워크 격리
- Public/Private 서브넷 분리
- 워커 노드 및 Pod는 Private 서브넷에만 배치
- NAT Gateway로 아웃바운드 트래픽 제어

---

## 개선 권장 사항

### 1. 보안 강화
- [ ] ALB 보안 그룹을 특정 IP 대역으로 제한 (현재: 0.0.0.0/0)
- [ ] WAF (Web Application Firewall) 적용
- [ ] AWS Shield Standard/Advanced 고려
- [ ] Secrets Manager로 민감 정보 관리

### 2. 모니터링 강화
- [ ] CloudWatch 로그 통합
- [ ] X-Ray 트레이싱 적용
- [ ] ALB 액세스 로그 활성화
- [ ] VPC Flow Logs 활성화

### 3. 비용 최적화
- [ ] NAT Gateway 대신 NAT Instance 고려
- [ ] 사용되지 않는 jenkins-master-tg 제거
- [ ] Reserved Capacity 검토

### 4. 성능 최적화
- [ ] CloudFront CDN 추가
- [ ] 정적 자산 S3 오프로드
- [ ] ALB 대상 그룹 스티키 세션 검토

### 5. 백업 및 재해 복구
- [ ] 다른 리전에 대기 클러스터 구성
- [ ] Route 53 Failover 라우팅 정책 적용
- [ ] EBS 스냅샷 자동화

---

## 부록

### 리소스 ID 참조표

#### VPC 및 서브넷
| 리소스 | ID | CIDR |
|--------|-------|------|
| VPC | vpc-09787158e8b6b2dca | 192.168.0.0/16 |
| Public Subnet 2d | subnet-0ac8af367a8c2f863 | 192.168.0.0/19 |
| Public Subnet 2a | subnet-0a60c17cc0939fb56 | 192.168.32.0/19 |
| Public Subnet 2c | subnet-098235b751ae8316a | 192.168.64.0/19 |
| Private Subnet 2d | subnet-09e95cfd32ae85443 | 192.168.96.0/19 |
| Private Subnet 2a | subnet-049b5be283190c06b | 192.168.128.0/19 |
| Private Subnet 2c | subnet-0b29f8571dcb7afc5 | 192.168.160.0/19 |

#### 보안 그룹
| 이름 | ID | 용도 |
|------|-------|------|
| ControlPlaneSecurityGroup | sg-0fd9cbbe5ade917d2 | Control Plane |
| EKS Cluster SG | sg-066ff3c2a7e86298f | Cluster ENI |
| ClusterSharedNodeSG | sg-024fd1f24408140ca | Worker Nodes |
| Shared Backend SG | sg-0bf801052c9c22014 | 로드 밸런서 공유 |
| trip-service ALB SG | sg-034f61e0a791bada2 | ALB 인그레스 |
| prometheus-grafana ALB SG | sg-089de9b761b99c72a | 모니터링 ALB |
| frontend NLB SG | sg-0c1508fbed215db21 | NLB 인그레스 |

#### 로드 밸런서
| 이름 | ARN (마지막 부분) | 타입 |
|------|------------------|------|
| trip-service-alb | 35fa0b2d3b66cb32 | ALB |
| k8s-monitori-promethe | efffcfa216330b4c | ALB |
| k8s-tripserv-servicef | 727230bd66326440 | NLB |

#### 타겟 그룹
| 이름 | ARN (마지막 부분) | 포트 | 타입 |
|------|------------------|------|------|
| k8s-tripserv-servicef-b3c3abeb59 | 3f9596448390fd49 | 80 | IP |
| k8s-tripserv-servicec-c81086e9b1 | 612f6fcf757a0b17 | 8000 | IP |
| k8s-tripserv-servicer-cc2fb486a0 | 98fe4bc5799b52dd | 8000 | IP |
| k8s-tripserv-serviceh-9dbd25740e | 192777768dbc17ce | 8000 | IP |
| k8s-tripserv-servicef-349dffcdbd | 0df2cf50d994f9e1 | 30839 | instance |
| k8s-monitori-promethe-2260d9f023 | 515b490c91ea76e3 | 3000 | IP |

### 문서 버전 정보
- **작성일**: 2025-10-20
- **최종 업데이트**: 2025-10-20
- **버전**: 1.0
- **작성 기준 데이터**: AWS CLI 조회 결과
