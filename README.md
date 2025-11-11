# K3s 홈 서버 클러스터

## 프로젝트 개요

**프로덕션급 K3s 클러스터**. 단순한 학습 환경이 아닌, 실제 외주 프로젝트를 운영하며 **GitOps, 모니터링, 로깅, CI/CD** 등 DevOps 핵심 역량을 실전에서 습득했습니다.

### 핵심 목표

- 실제 서비스 운영을 통한 DevOps 실무 경험 축적
- 클라우드 네이티브 기술 스택 학습 및 적용
- 고가용성(HA) 인프라 설계 및 운영

### 운영 중인 서비스

- **HM 프로젝트**: B2B 관리자 시스템 + B2C 클라이언트 (Dev/Prod 환경 분리)
- **목형 프로젝트**: 프로덕션 환경 운영 중

---

## 기술 스택

### Infrastructure

- **Kubernetes**: K3s v1.33.5 (경량 프로덕션 환경)
- **가상화**: LXD (3 Control-plane + 2 Worker 노드)
- **네트워킹**: MetalLB (Layer 2), Nginx Ingress, Tailscale VPN
- **HA 구성**: kube-vip (API Server VIP), etcd 3-node cluster

### DevOps & Observability

- **GitOps**: ArgoCD (9개 Application 관리)
- **CI/CD**: GitHub Actions + ArgoCD 연동
- **모니터링**: Prometheus + Grafana (kube-prometheus-stack)
- **로깅**: EFK Stack (Elasticsearch, Fluent Bit, Kibana)
- **인증서 관리**: cert-manager + Let's Encrypt

### 보안 & 접근 제어

- **VPN**: Tailscale (서브넷 라우팅으로 K8s API/SSH 접근 제어)
- **SSL/TLS**: 모든 Ingress에 자동 HTTPS 적용
- **도메인**: Cloudflare DNS 관리

---

## 인프라 아키텍처

### 클러스터 구성

```
┌─────────────────────────────────────────────────────────────┐
│                    Tailscale VPN Layer                       │
│          (K8s API, SSH 접근 - 관리자 전용)                    │
└─────────────────────────────────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────┐
│              kube-vip VIP: 192.168.0.160                     │
│         (apiserver.dog-foot.home - HA Endpoint)              │
└─────────────────────────────────────────────────────────────┘
                              ▼
┌───────────────────┬──────────────────┬─────────────────────┐
│   Control-plane   │  Control-plane   │   Control-plane     │
│      cp-a         │      cp-b        │       cp-c          │
│   (2vCPU/4GB)     │   (1vCPU/3GB)    │   (3vCPU/6GB)       │
│  etcd member 1    │  etcd member 2   │   etcd member 3     │
└───────────────────┴──────────────────┴─────────────────────┘
                              ▼
┌───────────────────────────────────────────────────────────┐
│                  Worker Nodes                              │
├─────────────────────────┬─────────────────────────────────┤
│     chaos-a             │         obs-b                   │
│  (1vCPU/2GB)            │     (2vCPU/8GB/120GB)           │
│  장애/부하 테스트 전용   │   모니터링/로깅 전용             │
└─────────────────────────┴─────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────┐
│              MetalLB (192.168.0.201-210)                     │
│          Nginx Ingress (192.168.0.201)                       │
└─────────────────────────────────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────┐
│           Public Internet (80/443 HTTPS Only)                │
│              *.dog-foot.com (Cloudflare DNS)                 │
└─────────────────────────────────────────────────────────────┘
```

### 물리 인프라

| 호스트 | CPU           | RAM  | 역할         | 배치 VM             |
| ------ | ------------- | ---- | ------------ | ------------------- |
| kkh    | Ryzen 5 3600  | 32GB | LXD 호스트 A | cp-a, cp-b, chaos-a |
| kkh2   | Ryzen 5 5600G | 32GB | LXD 호스트 B | cp-c, obs-b         |

---

## 주요 성과 및 기술적 도전

### 1. 고가용성(HA) 클러스터 구축

**도전 과제**: 단일 장애점(SPOF) 제거 및 무중단 운영 환경 구축

**해결 방법**:

- 3-node etcd 클러스터로 컨트롤 플레인 HA 구성
- kube-vip를 이용한 API Server VIP 자동 페일오버
- 2대의 물리 서버에 VM 분산 배치로 물리 장애 대응

**성과**: 컨트롤 플레인 1대 장애 시에도 서비스 무중단 운영 가능

### 2. GitOps 기반 선언적 배포 체계 구축

**도전 과제**: 수동 배포의 휴먼 에러 최소화 및 인프라 코드화

**해결 방법**:

- ArgoCD로 9개 Application 관리 (Dev/Prod 환경 분리)
- GitHub Actions에서 이미지 빌드 → Manifest 레포 변경 → ArgoCD 자동 배포
- Git을 통한 모든 변경 이력 추적 가능

**성과**: 배포 시간 단축 및 환경 간 일관성 보장

### 3. 통합 관찰성(Observability) 구현

**도전 과제**: 분산 시스템의 가시성 확보 및 장애 대응 시간 단축

**해결 방법**:

- **메트릭**: Prometheus + Grafana로 클러스터/파드 메트릭 실시간 모니터링
- **로그**: Fluent Bit + Elasticsearch + Kibana로 중앙 집중식 로그 수집
- **이벤트**: Kubernetes 이벤트 별도 인덱싱 (`k3s-events-*`)

**성과**: 장애 원인 분석 시간 대폭 단축, 사전 알림 설정 가능

### 4. 보안 및 네트워크 설계

**도전 과제**: 홈 네트워크 환경에서 안전한 관리 및 서비스 제공

**해결 방법**:

- Tailscale VPN으로 K8s API/SSH 접근 제한 (관리자 전용)
- cert-manager로 모든 서비스에 자동 HTTPS 적용
- MetalLB + Nginx Ingress로 외부 노출은 80/443만 제한

**성과**: 보안 위협 최소화하며 Let's Encrypt 인증서 자동 갱신

### 5. 리소스 최적화 및 전용 노드 운영

**도전 과제**: 제한된 리소스에서 모니터링 스택과 애플리케이션 분리

**해결 방법**:

- 워커 노드 역할 분리: `obs-b` (모니터링 전용), `chaos-a` (부하 테스트 전용)
- NodeSelector/Toleration으로 파드 배치 제어
- Elasticsearch 스토리지 최적화 (7일 주기 로그 정리)

**성과**: 메모리 부족 이슈 해결 및 안정적인 모니터링 운영

---

## 네임스페이스 및 서비스 구조

```
argocd            → GitOps 관리 (ArgoCD Server, ApplicationSet)
├── argocd.dog-foot.com

hm-dev            → HM 프로젝트 개발 환경
├── hm-api-dev.dog-foot.com
├── hm-admin-dev.dog-foot.com
├── hm-bong-client-dev.dog-foot.com
└── hm-playduk-dev.dog-foot.com

hm-prod           → HM 프로젝트 프로덕션 환경
├── hm-api-prod.dog-foot.com
├── hm-admin-prod.dog-foot.com
├── hm-bong-client-prod.dog-foot.com
└── hm-playduk-prod.dog-foot.com

mok-hyung-prod    → 목형 프로젝트 프로덕션
└── mok-prod.dog-foot.com

monitoring        → Prometheus + Grafana
└── grafana.dog-foot.com

logging           → EFK Stack
└── kibana.dog-foot.com

cert-manager      → SSL/TLS 자동화
ingress-nginx     → Ingress Controller
metallb-system    → LoadBalancer
```

---

## CI/CD 워크플로우

```
┌──────────────┐         ┌──────────────┐         ┌──────────────┐
│   개발자     │ ──Push→ │    GitHub    │ ──Build→│   ghcr.io    │
│  Code Push   │         │   Actions    │         │ (Container   │
└──────────────┘         └──────────────┘         │  Registry)   │
                                                   └───────┬──────┘
                                                           │
                         ┌─────────────────────────────────┘
                         │ Update Image Tag
                         ▼
                  ┌──────────────┐
                  │   Manifest   │
                  │  Repository  │ ◄─── GitOps 레포
                  └──────┬───────┘
                         │ Sync
                         ▼
                  ┌──────────────┐
                  │    ArgoCD    │ ◄─── Auto Sync
                  └──────┬───────┘
                         │ Deploy
                         ▼
                  ┌──────────────┐
                  │ K3s Cluster  │ ◄─── Dev/Prod 환경
                  └──────────────┘
```

---

## 학습 성과 및 역량

### 인프라 자동화

- Infrastructure as Code 원칙 적용 (모든 설정이 Git으로 관리)
- Helm Chart를 이용한 패키지 관리
- 선언적 매니페스트 작성 및 버전 관리

### 컨테이너 오케스트레이션

- Kubernetes 리소스 이해 (Deployment, Service, Ingress, PVC 등)
- 리소스 최적화 및 노드 스케줄링
- 네임스페이스 기반 멀티 테넌시

### 관찰성 구축

- 메트릭, 로그, 트레이스의 3 Pillar 이해
- PromQL을 이용한 쿼리 작성 및 대시보드 구성
- 중앙 집중식 로깅 시스템 운영

### 네트워크 및 보안

- CNI, Service Mesh 개념 이해
- Ingress Controller 및 LoadBalancer 운영
- TLS 인증서 자동화 및 VPN 기반 접근 제어

### 문제 해결 경험

- etcd 클러스터 장애 복구
- 리소스 부족으로 인한 파드 스케줄링 실패 해결
- DNS 및 네트워크 정책 디버깅

---

## 향후 계획

- [ ] Helm Chart 기반 애플리케이션 패키징
- [ ] Kubernetes Operator 개발 학습
- [ ] Service Mesh (Istio/Linkerd) 도입
- [ ] Chaos Engineering 실습 (chaos-a 노드 활용)
- [ ] Backup & Disaster Recovery 자동화
- [ ] Prometheus Alertmanager 알림 규칙 고도화

---

## 프로젝트 정보

- **GitOps 레포지토리**: [dog-foot-k8s-manifests](https://github.com/Team-DogFoot/dog-foot-k8s-manifests)
- **클러스터 구성**: 3 Control-plane + 2 Worker
- **관리 도구**: kubectl (MacBook에서 Tailscale VPN 경유)
- **운영 기간**: 2024.09 ~ 현재

---

## 연락처

**이 프로젝트에 대해 더 자세히 알고 싶으시거나 질문이 있으시다면 언제든 연락 주세요.**

---

> 이 프로젝트는 단순히 기술을 학습하는 것을 넘어, 실제 서비스를 운영하며 **안정성, 확장성, 보안**을 고민하고 해결한 DevOps 실전 경험의 결과물입니다.
