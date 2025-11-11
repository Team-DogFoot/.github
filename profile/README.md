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

## 향후 계획

- [ ] Helm Chart 기반 애플리케이션 패키징
- [ ] Service Mesh (Istio/Linkerd) 학습
- [ ] 장애 상황 대응 훈련 (chaos-a 노드 활용)
- [ ] Backup & Disaster Recovery 자동화
- [ ] Prometheus Alertmanager 알림 규칙 고도화
