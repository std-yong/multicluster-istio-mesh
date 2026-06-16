# multicluster-istio-mesh

Kubernetes 멀티클러스터를 Istio Service Mesh로 연결하고, 그 위에 **mTLS · 카나리 배포 · 서킷브레이커**를 인프라 레벨에서 적용했던 학습 프로젝트 정리.

> **이 레포에 대하여**
> 2023년 카카오클라우드스쿨 후반에 진행한 팀 프로젝트(3인)의 운영 사고와 매니페스트를 정리한 docs 중심 레포입니다.
> 원본 클러스터 매니페스트는 학원 환경 종료와 함께 사라졌고, 본 레포의 `examples/`는 당시 운영했던 설정을 본인 기록과 회고를 기반으로 **재구성**한 것입니다. 실행 가능한 IaC 묶음이 아니라 "어떤 설정으로 어떤 문제를 풀었는가"를 보여주는 자료입니다.

> 시연 영상은 학원 LMS에만 남아 있어 공개 링크가 없습니다.

---

## 한 줄 요약

K8s 단일 클러스터의 SPOF 문제를 **멀티클러스터 + Istio Service Mesh로 해결**하고, **카나리 가중치 배포와 Outlier Detection 기반 서킷브레이커**를 매니페스트만으로 적용. CI는 Jenkins, CD는 ArgoCD GitOps, 모니터링은 4 Golden Signals 기준.

---

## 디렉토리

| 경로 | 내용 |
|---|---|
| `examples/multicluster/` | Remote Secret 교환, IstioOperator 멀티클러스터 설정 |
| `examples/traffic/` | DestinationRule + VirtualService (카나리 90:10) |
| `examples/resilience/` | Outlier Detection 기반 서킷브레이커 |
| `scripts/` | 멀티클러스터 셋업 명령 스크립트 |
| `docs/architecture.md` | 멀티클러스터 토폴로지, 4 Golden Signals 대시보드 |
| `docs/incidents.md` | PKI 한계로 범위 조정한 회고 + 임계값 결정 근거 |

---

## 주요 결정

- **단일 클러스터 SPOF → 멀티클러스터**: 같은 mesh ID(`mesh1`)를 공유하는 Primary-Remote 구조 + Remote Secret 교환으로, 한쪽 Control Plane이 죽어도 다른 쪽이 받아주는 토폴로지.
- **다른 네트워크 멀티클러스터 포기 → 동일 네트워크로 범위 조정**: East-West Gateway TLS handshake 실패 → 공통 Root CA 기반 PKI가 필요하다는 결론. 학습 시간 내 PKI 자동화까지 못 짠다고 판단해 범위 축소, 회고로 명시. → [docs/incidents.md](docs/incidents.md)
- **Spinnaker → ArgoCD**: Halyard가 8GB VM에서 OOM. 메모리 문제 자체보다 *Git이 단일 진실의 원천*이 되는 GitOps 모델이 K8s 선언적 관리와 정렬되어 갈아탐.
- **카나리는 코드 수정 없이**: VirtualService 가중치만으로 `90:10 → 70:30 → 50:50 → 0:100` 단계적 롤아웃. 에러율 튀면 즉시 0으로 되돌리는 회피 경로 확보.

---

## 서킷브레이커 임계값 결정 근거

| 파라미터 | 값 | 이유 |
|---|---|---|
| `consecutiveGatewayErrors` | 5 | 1~2회는 일시적 네트워크 지연과 구분 불가 → 오탐 |
| `interval` | 10s | 너무 짧으면 정상 회복 중인 서비스를 반복 격리 |
| `baseEjectionTime` | 30s | 재시작·자동 복구에 충분한 여유 |
| `maxEjectionPercent` | 50% | 전체 격리 시 서비스 다운 → 절반은 트래픽 유지 |

> 임계값은 *기능*이 아니라 *정책*. 5/10s/30s/50% 모두 트레이드오프이고, 그 근거를 매니페스트 옆에 같이 적어두지 않으면 후임이 이유 없이 값을 바꾼다.

---

## 4 Golden Signals 모니터링

장애 시 "어디부터 볼까"를 분명히 하기 위해 Google SRE 4 Golden Signals 기준으로 항목 분리.

| Signal | 메트릭 | 의미 |
|---|---|---|
| Latency | `istio_request_duration_milliseconds` (p50/p95/p99) | p99 튀면 일부 사용자 영향 |
| Traffic | `istio_requests_total` (req/s) | 스케일아웃 판단 기준 |
| Errors | `istio_requests_total{response_code=~"5.."}` | 서킷브레이커 트리거와 연계 |
| Saturation | CPU/Memory, Connection Pool | 한계 도달 전 선제 대응 |

대시보드는 **Mesh(사용자 관점)**과 **Control Plane(운영자 관점)**으로 분리해 "서비스 장애"와 "Istio 자체 장애"를 다른 대응 경로로 처리.

---

## 회고 / 남은 과제

- **PKI 운영 미완성** — 다른 네트워크 멀티클러스터를 끝까지 끌고 가지 못했음. cert-manager + 공통 Root CA, 또는 클라우드 매니지드 CA(AWS ACM-PCA, GCP CAS)로 재도전이 다음 과제
- **부하 테스트 부재** — 4 Golden Signals 임계값을 업계 일반치로 잡았고, k6/JMeter 실측은 미진행
- **Helm 미적용** — Raw YAML로 학습 우선. 환경 분리는 Helm + ArgoCD ApplicationSet이 다음 단계
- **HPA 미도입** — Prometheus Adapter + 커스텀 메트릭 기반 HPA로 트래픽 급증 시 자동 스케일아웃 검증 필요

## 지금 다시 한다면

- Istio Ambient Mesh (사이드카리스) 검토 — 사이드카 리소스 오버헤드 제거
- 공통 Root CA + cert-manager로 PKI 자동화부터 깔고 멀티클러스터 시작
- 카나리 배포는 Argo Rollouts와 결합 (가중치 자동 progression + 메트릭 기반 자동 롤백)
